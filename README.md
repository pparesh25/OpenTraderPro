# OpenTrader Pro

> A desktop-first multi-broker trading + analytics platform for Indian equity, F&O, commodity, and global crypto markets. Built in Python + Qt 6. **Local-first.** Plugin-extensible. Currently in the final phase of development + testing — beta-tester recruitment opens with the v0 release.

|  |  |
|---|---|
| **Status** | Pre-beta hardening (V3 plugin architecture + R3 review closed) |
| **Beta target** | 2026 — exact date confirmed once UI polish + remaining hardening land |
| **Source** | Private during dev/test; public release pending license decision |
| **Marketplace** | [Public](https://github.com/pparesh25/OpenTraderPro-MarketPlace) — 28 reviewable plain-Python files (5 plugins + 3 strategies + 20 indicators) |
| **Platforms** | macOS, Windows, Linux |
| **Stack** | Python 3.13+, PySide6 (Qt 6.10), pyqtgraph, DuckDB |

---

## 📸 Screenshots

> _Screenshots are being captured during the UI polish pass. Placeholders below mark the planned set; this section will fill in before beta._

```
[ Screenshot 1 — Main window: chart tab + watchlist + indicator panel + status bar ]
[ Screenshot 2 — Multi-chart layout: 4 tabs, two with 1M intraday + two daily, sub-panels visible ]
[ Screenshot 3 — Drawing tools palette + Fibonacci + trendline rendered on a candle chart ]
[ Screenshot 4 — Trading Hub: Quick Order tab with pre-flight tunnel-status warning ]
[ Screenshot 5 — Trade Journal: tax-breakdown columns (STT / GST / SEBI charges / stamp duty) + CSV export ]
[ Screenshot 6 — Strategy Panel: schedule editor + per-strategy live log + multi-account targets ]
[ Screenshot 7 — Backtest Panel: equity curve + trade list + metrics + CSV export ]
[ Screenshot 8 — CSV import wizard: column-mapping UI for a non-standard date+time CSV ]
[ Screenshot 9 — Settings → Marketplace: developer-mode toggle + alt-marketplace URL ]
[ Screenshot 10 — Settings → Routed plugins: per-plugin tunnel-status indicator dots ]
```

---

## What's actually inside

The sections below are written from a deep code tour — concrete file types, exact formats, table names, hard limits. If a feature is partial or planned, it's flagged.

---

### 1. Data import — DuckDB + flexible CSV

The app stores every bar / tick / order / fill in a **DuckDB** file on your disk (default location `~/.opentrader-pro/`, configurable per database). DuckDB is the embedded analytical database from the same family as PostgreSQL — single-file, columnar, vectorised, no server. Bulk-load 1M+ rows in seconds without leaving Python.

**Schema:**
- Primary table `price_data` — columns: `symbol, date, time, timeframe, open, high, low, close, volume, open_interest`, plus three reserved `extra_1/2/3` slots for series-specific metadata (delivery quantity, total trades, series code).
- Constraints: `open/high/low/close > 0`, `high >= low`, `volume >= 0` — bad rows are filtered at the SQL level, not in Python loops.

**CSV import wizard** (`opentrader/services/import_engine.py` + `import_constants.py`):
- **Delimiters auto-detected** — comma, semicolon, tab, pipe.
- **Headers auto-detected** with broad alias coverage:
  - Symbol: `symbol`, `ticker`, `sym`, `scrip`, `stock`, `name`, `sc_name`
  - Date: `date`, `dt`, `trade_date`, `trading_date`, `timestamp`
  - Time: `time`, `tm`, `trade_time`
  - OHLC: `open/high/low/close`, plus `open_price`, `ltp`, `closing`
  - Volume: `volume`, `vol`, `v`, `qty`, `quantity`, `turnover`, `tottrdqty`
  - Open interest: `oi`, `open_interest`, `openinterest`, `oi_contracts`
  - Extras (mapped to `extra_1/2/3`): `series`, `total_trades`, `delivery_qty`
- **Date formats — 7 auto-detected:** `%Y-%m-%d`, `%Y%m%d`, `%d-%m-%Y`, `%m/%d/%Y`, `%d/%m/%Y`, `%Y/%m/%d`, `%d-%b-%Y`.
- **Time formats — 11 presets:** `None (EOD)`, `HH:MM`, `HH:MM:SS`, `HHMM`, `HHMMSS`, `HH.MM`, `HH-MM`, `HH:MM:SS.fff`, `HH:MM AM/PM`, two more.
- **Manual column mapping** — when auto-detect can't, users assign columns by index in the wizard.
- **Duplicate handling** — modes `update` (replace) or `skip` (preserve existing). Per-batch.
- **Speed** — pure DuckDB SQL path: `read_csv_auto()` → staging table → `INSERT OR REPLACE` with validation predicate. Typical 10k–50k rows/sec depending on disk.
- **Batch import** — up to 200 files per batch; UI progress bar + cancel button.

**The point:** point the wizard at your historical data zip from any vendor (NSE bhavcopy, MCX archive, Binance kline export, your own broker downloads) — the importer figures out the shape, skips bad rows with a count, and the imported series is immediately chartable + backtestable.

`[ Screenshot: CSV import wizard — column mapping ]`

---

### 2. High-performance charting — 1M data, native pan/zoom

Charting is built on **pyqtgraph** (pure-Python over Qt, no browser, no JS). The price panel shipped with optimisations specifically for 1-minute series with millions of bars:

- **Tile-based rendering** — `CandlestickItem` extends a custom `TiledGraphicsItem` that chunks the data into viewport-sized tiles so the renderer never touches off-screen bars.
- **Viewport clipping** — `setClipToView(True)` on every `PlotItem`; pan/zoom only invalidates what actually moved.
- **Auto-decimation for line charts** — peak-preserving downsampling kicks in past a configurable threshold (`setDownsampling(auto=True, method='peak')`).
- **NumPy-backed buffer** — `BarBuffer` keeps OHLCV in raw NumPy arrays; the DataFrame is materialised only on the first `to_dataframe()` call, then cached.
- **Optional OpenGL acceleration** — set `OPENTRADER_OPENGL=1` to enable hardware-accelerated rendering (off by default for cross-platform consistency).
- **Custom `TimeAxis`** — date-aware tick formatting; weekend/holiday gaps suppressed (no flat empty stretches).
- **Per-visible-bar Y-auto-range** — pan to a different stretch and the price scale snaps to that stretch's actual range, not the full series.

Result: 1M bars across 6+ months pan/zoom smoothly on a modest laptop; intraday sessions on equity (~390 bars per day × multiple years) render imperceptibly. The frame budget never balloons just because the dataset got bigger.

`[ Screenshot: 1M candle chart with smooth pan + sub-panel indicators ]`

---

### 3. Drawing tools — 12 types, persisted per (symbol, timeframe)

Drawings live on the chart, are saved to DuckDB, and reload exactly where you left them — even after reopening the app, switching tabs, or changing timeframes (drawings are scoped per `(symbol, timeframe)`).

**Tools shipped (12):**
| Tool | Purpose |
|---|---|
| `line` | trendline (two anchor points) |
| `horizontal_line` | price-level horizontal |
| `vertical_line` | time-level vertical |
| `ray` | semi-infinite trendline |
| `rectangle` | range box |
| `circle` | range ellipse |
| `channel` | parallel-line trend channel |
| `fibonacci` | retracement / extension levels |
| `pitchfork` | Andrews' pitchfork (3 anchors) |
| `gann_fan` | Gann fan angles |
| `price_range` | price-range measure with delta + % display |
| `text` | annotation label |

**Persistence:**
- DB table: `chart_objects` — columns `id, symbol, type, properties (JSON), created_at, updated_at`.
- Drawing model serialises every anchor point + style (color, width, dash) + extras (timeframe, z-order, locked, visible flags).
- `DrawingService` provides `create / load / update / delete` with Qt signals (`drawing_added`, `drawing_removed`, `drawing_updated`, `drawings_loaded`) so multiple tabs of the same symbol stay in sync.
- Per-symbol in-memory cache so re-rendering a tab doesn't re-query the DB.

**Edit:** `DrawingController` runs a small state machine — IDLE → DRAWING → IDLE on first-paint, or → SELECTED → MOVING → IDLE on grab + drag of an existing drawing.

`[ Screenshot: Fibonacci + trendline + rectangle on chart ]`

---

### 4. Drag-and-drop indicators — 20+ ship, more in 30 lines of Python

Twenty indicators ship today via the marketplace as plain `.txt` Python files; you drag them from the indicator panel and drop them onto a chart pane.

**Shipped indicators:**
- **Averages (7):** SMA, EMA, DEMA, TEMA, WMA, VWAP (intraday session-anchored), Parabolic SAR.
- **Bands (3):** Bollinger Bands, Keltner Channels, Ichimoku Cloud.
- **Oscillators (10):** RSI, MACD, Stochastic, ADX, Aroon, ATR, CCI, MFI, OBV, Williams %R.

**Drop targets:**
- **Drop on price panel** → indicator becomes an **overlay** on the candle chart, with its own right-side Y-axis (e.g. SMA + Bollinger over the candles).
- **Drop on a sub-panel sheet** → indicator gets a dedicated panel below the price (e.g. RSI in its own pane with its own zoom).
- MIME format is plain text `"category:name"` (e.g., `oscillators:RSI`); makes copy-paste programmatic-add equally easy.

**Author your own:** drop a `.txt` into `~/.opentrader-pro/indicators/{category}/`, restart, the indicator appears in the panel. The `BaseIndicator` contract is ~30 lines — `compute(df)` returns a DataFrame of series; the renderer figures out the rest.

**Per-tab clone management** (`IndicatorSync`) — one master list of indicators applies to all tabs by default, with each tab keeping its own clone for parameter tweaks. Changing a master parameter syncs to every tab; changing a tab's local clone is isolated.

`[ Screenshot: Indicator tree → drop SMA on price + drop RSI on a sheet ]`

---

### 5. Watchlists — `.txt` files, AmiBroker-compatible, config-aware

Watchlists are plain text files (one symbol per line), readable in any editor and compatible with AmiBroker's `.TLS` format. The first line of the file is an optional **config comment**:

```
# zerodha::default, NSE, NSE_EQ, 5m, 30day, live=true
RELIANCE
TCS
INFY
HDFCBANK
```

**Config fields:**
- `plugin::alias` — the v2 account ID this watchlist subscribes through (e.g. `mock::main`, `binance::spot`).
- `exchange` — broker exchange code (e.g. `NSE`, `NASDAQ`).
- `segment` — market segment for hours/holiday lookup (e.g. `NSE_EQ`, `MCX_NONAGRI`, `CRYPTO_SPOT`).
- `timeframe` — canonical code (`1m`, `5m`, `15m`, `1h`, `D`, `W`, `M`).
- `backfill_days` — how much history to fetch on first authentication.
- `live=true/false` — subscribe to real-time ticks after backfill.

**Storage:** `<database_dir>/watchlists/<watchlist_name>.txt`.
**Multi-config support:** different watchlists can target different brokers, exchanges, segments, and timeframes — each is independent.
**Legacy 5-field shape** (without `segment`) is still parsed; `WatchlistService.save_watchlist()` rewrites in the latest 6-field form.

`[ Screenshot: Watchlist panel + config dialog ]`

---

### 6. Symbol management — merge

The Symbol Manager dialog lets you reconcile duplicates that creep in across data sources (e.g. Yahoo's `RELIANCE.NS` vs your bhavcopy import's `RELIANCE`).

**Merge** (`SymbolService.merge_symbols(source, target, *, keep_source=False)`):
- Source rows on dates the target doesn't have are copied over.
- Overlapping dates keep the target's data (no averaging).
- Drawings are moved with the data.
- Every watchlist file is rewritten to replace `source` references with `target`.
- `keep_source=False` deletes the source symbol + all its remaining data after the merge.

> **Note: split is in design, not yet implemented.** The placeholder exists in the service layer for the eventual stock-split / corporate-action workflow but is not exposed in the UI today.

`[ Screenshot: Symbol Manager merge dialog ]`

---

### 7. Multiple chart tabs + session save/load

A chart tab is a self-contained workspace: one symbol, one timeframe, multiple indicator panels, your drawings. Open up to **20 chart tabs** in parallel.

**Layout per tab:**
- Top 70%: price panel (candles/bars/line + overlay indicators + drawings).
- Bottom 30%: an 8-sheet `QTabWidget` —
  - Sheet 1 = **volume** (always-on, locked Y-zoom).
  - Sheets 2–8 = oscillator slots (RSI / MACD / Stochastic / ATR / etc., one per sheet).
- Splitter ratio + active sheet are remembered per tab.

**Session persistence** — every tab's state is serialised to `<database_dir>/app_state.json`:
- Symbol, exchange, timeframe.
- Chart viewport (`x_range`, per-sheet Y-zoom ratios, splitter sizes).
- Active sheet index.
- Each indicator's state (placement, cloned-from master, parameter overrides).

On next launch the app re-opens the same tabs with the same viewports + the same indicators positioned exactly where you left them. Lazy-load: a tab doesn't fetch data until you switch to it for the first time.

`[ Screenshot: 4 chart tabs open, two daily, two intraday ]`

---

### 8. Trading Hub — five tabs

The Trading Hub is a separate, non-modal floating window that remembers its position, size, and active tab. Five tabs:

**1. Accounts**
- Lists every registered broker account (data + exec plugins).
- Per-account login/logout with OAuth flow handlers, balance, holdings, margins.
- "Install marketplace files" button → fetches signed plugins/strategies/indicators from the marketplace.
- Per-plugin tunnel routing indicator (green = SOCKS5 tunnel up, red = down).

**2. Market Watch**
- Live tick table — symbol, last price, bid/ask, volume.
- Click a row → switches the active chart tab to that symbol.

**3. Trading**
- **Quick Order** (top half): symbol/exchange/segment dropdowns, side (Buy/Sell), quantity, order type (MARKET / LIMIT / SL / SL-M), product (CNC / MIS / NRML — capability-aware per broker), price, place/cancel. Pre-flight tunnel-status warning if a routed broker would attempt an order with the SOCKS5 tunnel down.
- **Order status / fill history** (bottom half): recent orders + fills across authenticated accounts.

**4. Strategy**
- Strategy list (marketplace + user dir).
- Per-strategy config: target watchlist, target accounts, parameters.
- Schedule editor: once / daily / bar-close-driven trigger modes with per-segment market-hours awareness + holiday gating.
- Per-strategy live log.

**5. Backtest**
- Watchlist + strategy picker.
- Date range, initial capital, max positions, commission/slippage/STT inputs.
- Equity curve + trade list + metrics + CSV export (see §12 below).

**Trade Journal** is a sibling panel (sometimes rendered as a 6th tab depending on the layout); see §13 for its capabilities.

`[ Screenshot: Trading Hub with 5 tabs visible ]`

---

### 9. Connectors — V2/V3 plugin contract

Every broker integration is a **plugin** — a plain `.txt` Python file the app loads at runtime. Two contracts (`opentrader/connectors_v2/contract.py`):

**`DataPluginV2`** (data + realtime):
- Class attributes — `INFO`, `CREDENTIAL_SCHEMA`, `CAPABILITIES` (segments, realtime, historical, search, exchange→segment map, rate limits, static-IP-required-for, websocket routing).
- Methods — `authenticate(creds) → AuthResult`, `fetch_historical(symbol, exchange, segment, start, end, timeframe)`, `search_symbols`, `subscribe_realtime` / `unsubscribe_realtime`, `on_tick`.

**`ExecPluginV2`** (orders + positions):
- Plus `ExecCapabilities` — product types, modify/cancel/OCO support, `is_simulation`, `async_execution_reports`.
- Methods — `place_order`, `cancel_order`, `modify_order`, `get_positions`, `get_holdings`, `get_margins`, `calculate_fees(fill) → Fees`, `get_market_hours(segment)`, `get_timezone()`, `test_connection`.
- Qt signal `execution_report` — push channel for fills (no polling lag).

**Built-in (source-shipped):**
- `MockExecPlugin` — instant fills at LTP, no auth, broker-agnostic sandbox.
- `BacktestExecPlugin` — walk-forward sim driver.

**Marketplace (signed `.txt` files):**
- `openbb_data` (multi-provider data), `kite_broker_data` + `kite_broker_exec` (Zerodha India), `crypto_exchange_data` + `crypto_exchange_exec` (Binance spot + USD-M + COIN-M).
- Verified at install + at every load against an embedded Ed25519 public key. CI on the marketplace repo verifies signatures on every PR.

**User-edit plugins:** `~/.opentrader-pro/{plugins,strategies,indicators}/` — gated by Developer-Mode toggle (off by default) + a one-time consent dialog per category.

`[ Screenshot: Accounts tab + plugin list with green/red status ]`

---

### 10. Watchlist-driven backfill + segment routing

When a data plugin authenticates, `AutoDataService.start(account_id, watchlists)` kicks off:

1. Filter all watchlists to those targeting the just-authenticated account.
2. For each symbol in those watchlists:
   - Query DB for the latest stored bar.
   - If empty → fetch the last `config.backfill_days`.
   - If present → fetch from `(last_stored_date + 1)` to today.
3. Deduplicate `(symbol, timeframe)` tasks.
4. Call `plugin.fetch_historical(...)` in a background thread; save the returned bars via `PriceRepository.save_ohlcv()`.
5. If `config.live=true` and the plugin supports realtime → subscribe; ticks flow into `RealtimeManager` and fan out to subscribed chart tabs.

**Segment routing:** each watchlist's `config.segment` is threaded through to the plugin so equity vs. F&O vs. commodity vs. crypto-spot calls hit the correct broker venue. Empty segment → inferred from the plugin's `Capabilities.exchange_to_segment` map.

**Progress signals:** `backfill_progress(current, total, symbol)` + `backfill_done()` flow into the main window's status bar.

`[ Screenshot: Status bar during backfill — "Fetching RELIANCE 5m (12/47)" ]`

---

### 11. Multi-account routing — different watchlists, different strategies, different brokers

This is the feature that makes the platform genuinely multi-broker rather than just "swap one for another".

The `SignalRouter` (`opentrader/connectors_v2/signal_router.py`) takes a `TradeSignal` from a strategy and fans it out across **multiple target accounts** in parallel — each account can be a different broker, a different alias of the same broker, or a paper-trading wrapper. Per-account gates run independently:

- **Risk check** — open positions vs `max_open_positions`.
- **Daily order check** — orders today vs `max_daily_orders`.
- **Market-hours gate** — current time vs the plugin's `MarketWindow` for the signal's segment (rolls on the plugin's local midnight — IST for Zerodha, UTC for Binance, etc.).
- **Holiday gate** — plugin baseline + user-override JSON; rejects on closed days.
- **Rate-limit gate** — sliding-window per plugin (Zerodha 10/sec + 3 queries/sec; Binance 50 orders / 10s).

**Risk limits live in `settings.json`:**

```json
{
  "risk_limits": {
    "mock::main":         {"max_daily_orders": 50,  "max_open_positions": 10},
    "zerodha::default":   {"max_daily_orders": 100, "max_open_positions": 5},
    "binance::spot":      {"max_daily_orders": 200, "max_open_positions": 20}
  },
  "default_accounts": ["mock::main"]
}
```

**Strategies declare their targets explicitly:**
- `ctx.target_brokers = ["zerodha::main", "binance::spot"]` for fan-out.
- Or `target_accounts=` kwarg on individual signals.
- Falls back to `default_accounts` if a signal arrives empty.
- Empty + no defaults → rejected with `"No target accounts"` (no implicit "active broker" magic).

**Result:** one strategy can simultaneously trade Indian equity on Zerodha and crypto on Binance, with separate risk caps and separate holiday calendars, all from one process. Add a paper-account into the same fan-out and you can A/B a real-money execution against a paper one with identical signals.

`[ Screenshot: Strategy → 3 target accounts simultaneously, two real + one paper ]`

---

### 12. Mock broker + paper trading + multi-session

**Mock broker** (`MockExecPlugin`) ships in-source — no auth, no broker, instant fills at the last traded price. The same plugin you'd use to teach a strategy framework class works as a sandbox for any logic you want to test without risking capital.

**Paper trading service** (`PaperTradingService`) wraps **any** real exec plugin:

```python
PaperTradingService.wrap(real_plugin, last_price_provider=..., alias=..., state_repo=...)
```

- Reads (`get_holdings`, `get_margins`, `get_market_hours`, `calculate_fees`) → delegate to the real plugin → real broker context, real fees, real holiday calendar.
- Writes (`place_order`, `cancel_order`, `modify_order`) → simulated in-memory.
- MARKET orders fill instantly at LTP; LIMIT / SL / SL-M orders sit pending and fill when ticks cross the trigger (real per-tick simulation, not just bar-close).
- **Optional persistence** — `PaperStateRepo` writes to `~/.opentrader-pro/paper_state.db`, so a paper session survives an app restart.

**Multi-session by design** — the `AccountRegistry` keys every authenticated session by `plugin::alias` (the `AccountId`). `zerodha::main` and `zerodha::family` are two independent sessions with separate credentials, separate keychain entries, separate session cookies, separate rate-limit windows. A strategy can target both at once. Crypto users can run `binance::spot`, `binance::usdm`, `binance::coinm` in parallel — each picks up its own session via `session_registry.get_or_create_session()`.

Switch any account between live and paper from the Settings → Trading tab: add the alias to `paper_account_aliases` and on next app start it's wrapped automatically.

`[ Screenshot: Settings → Paper accounts list editor ]`

---

### 13. Strategy backtest — same code path as live, CSV report

The `BacktestEngine` runs **the same `Strategy.run(ctx)` body** that powers live execution. No "backtest engine with subtle differences" — the context is a `BacktestContext` that slices historical data up to the simulation time, exposes the same indicator helpers and signal factories, and feeds the same `SignalRouter` shape.

**Model:**
- Signal at close of bar T → fill at open of bar T+1 (zero lookahead).
- Multiple open positions per symbol allowed (pyramid / scale-in strategies).
- Equal-weight slot allocation across `max_positions`.
- Commission + slippage on entry; commission + slippage + STT on exit.
- Product-aware tax rates (CNC 0.10 % vs MIS 0.025 % vs NRML 0.0125 % for Indian STT).

**`BacktestResult` dataclass:**
- `strategy_name, watchlist_name, timeframe, start_date, end_date, initial_capital, final_equity`.
- `trades: list[TradeRecord]` — per trade: symbol, entry/exit date+price, quantity, P&L, P&L %, commission, STT, status, exit_reason.
- `equity_curve: pd.Series`.
- `metrics: dict` — total return, Sharpe (252-day or asset-specific), max drawdown, win rate, profit factor, average win/loss, etc.
- `daily_log: list[DailySnapshot]` (when `enable_daily_log=True`) — per-day cash, portfolio value, signals generated, signals skipped, opened today, closed today.
- `halt_log: dict[str, list[Timestamp]]` — bars dropped due to NaN OHLC (suspected halt) so the strategy author can spot data gaps in the result.

**International defaults** — STT default is 0.0, commission default is 0.0, risk-free-rate default is None. Indian-equity users opt-in by passing the named `INDIA_*` constants. No phantom Indian taxes on a US-equity or crypto backtest.

**CSV export** from the Backtest panel: trades table (every TradeRecord field flat-columned) + metrics row + equity-curve series.

**Cancellable mid-bar:** strategies can poll `ctx.check_cancel()` between heavy ops; the Cancel button responds within milliseconds even on a long-running indicator computation.

`[ Screenshot: Backtest panel — equity curve + trade list + metrics ]`

---

### 14. Trade journal — full tax breakdown, push-channel updates

Every signal, every order, every fill, every position snapshot — recorded in DuckDB the moment it happens. No polling lag. Five v2 tables (suffix is intentional — coexist with the legacy v1 schema during the transition):

| Table | Contents |
|---|---|
| `trade_signals_v2` | Every TradeSignal received, one row per (account, signal). |
| `trade_orders_v2` | Every order placed; status transitions tracked (PENDING → PARTIAL → COMPLETE / CANCELLED / REJECTED). |
| `trade_positions_v2` | Open positions snapshot per account + symbol. |
| `trade_pairs_v2` | Matched entry/exit pairs — for FIFO/LIFO realised-P&L analysis. |
| `trade_cash_snapshots_v2` | Daily cash + equity snapshots. |

**Push-channel updates** — exec plugins emit a Qt `execution_report` signal on each fill (V3 §F7). The journal listens directly; no 5-second polling window where a fill is invisible.

**Per-fill fees** — every order's `fees` column stores the plugin's full breakdown as JSON. India brokers populate the flat shadow columns automatically: `brokerage`, `exchange_charges`, `STT`, `CTT`, `GST`, `SEBI charges`, `stamp duty`. Crypto brokers populate maker/taker rate columns. Unknown keys stay in the JSON blob — no schema change to support a new fee shape.

**MTM updater** — a 60-second-default cadence (configurable in Settings → Trading) recomputes unrealised P&L for open trade pairs. Power users on tight rate limits can dial it up; scalpers can dial it down to 10–15 s.

**CSV export** — date range filter, account filter, symbol filter, status filter (open / closed / both), then export to CSV with flattened tax columns. Per-financial-year filtering ships pre-configured for Indian users (Apr–Mar fiscal year).

`[ Screenshot: Trade Journal panel — flat tax columns + CSV export button ]`

---

## 🏗️ Architecture at a glance

| Layer | Tech |
|---|---|
| UI | PySide6 (Qt 6.10) — native desktop, no Electron, no browser |
| Charting | pyqtgraph (pure Python over Qt; optional OpenGL acceleration) |
| Language | Python 3.13+ (works on 3.10+) |
| Persistence | DuckDB (single-file embedded analytical DB) |
| Plugin loading | `compile()` + `exec()` of Ed25519-signed `.txt` files |
| Background work | Qt signals + `QueuedConnection` — broker reactor threads never touch the UI thread |
| Routing | `socks5h://` (DNS-on-remote) for SEBI compliance; live-resolver pattern degrades gracefully when tunnel drops |
| Schedule | APScheduler with per-plugin timezone awareness (graceful fallback if the dependency is missing) |
| Distribution | Pure Python today; optional Nuitka for binary release |

**Local-first.** No mandatory cloud account. No telemetry. Broker keys in your OS keychain (macOS Keychain / Windows Credential Manager / Linux Secret Service). Trades, orders, fills, drawings, sessions — all in your DuckDB file.

**Plugin-extensible.** Source ships only `mock` + `backtest`. Real brokers, strategies, and indicators come from the [marketplace](https://github.com/pparesh25/OpenTraderPro-MarketPlace) as plain-text reviewable Python.

**Codebase shape:** ~290 Python modules across `connectors_v2/`, `data/`, `drawing_tools/`, `indicators/`, `models/`, `services/`, `strategy/`, `ui/`, `viewmodels/`, `utils/`, `tests/`. ~89k LOC. Pytest suite of 2800+ tests gates every PR.

---

## 🛒 Marketplace

The companion repository [**OpenTraderPro-MarketPlace**](https://github.com/pparesh25/OpenTraderPro-MarketPlace) (public since 2026-04-29) hosts the canonical plugins, strategies, and indicators as plain `.txt` Python — 28 reviewable files in a strict directory layout. Every file ships with an Ed25519 `.sig` sibling; the app verifies signatures at load. CI on the marketplace verifies signatures on every PR + cross-checks the embedded public key against the app repo (drift detection).

Read the marketplace's [README](https://github.com/pparesh25/OpenTraderPro-MarketPlace) for the philosophy: source-readable as the primary defence, signing as defence-in-depth, **user review of downloaded files as a shared responsibility**.

---

## 🚦 Development status

**Final hardening phase.** Closed in 2026-04 over a multi-laptop coordinated review (R3):

- ✅ V3 plugin architecture (5 phases: F1-F9, N1-N4, P1-P5, R1-R4)
- ✅ Real-time `executionReport` push channel (V3 §F7)
- ✅ Marketplace cryptographic signing — Ed25519 keypair, embedded public key, GitHub Action verifies on every push, drift detection both directions (V3 §M1)
- ✅ Trusted-only mode by default + Developer-Mode toggle (V3 §M2)
- ✅ In-app marketplace installer (V3 §M3)
- ✅ Strategy + indicator consent gate (V3 §S3)
- ✅ Marketplace public flip (Phase M4, 2026-04-29)
- ✅ R3 review cycle complete — 22 critical + 64 major + 26 minor findings closed across ~30 PRs; remaining 75 findings are opportunistic minors

🚧 **Now:** UI polish + screenshot pass + license decision + beta launch.

**Source code is not yet publicly available.** License decision is in progress; the marketplace ships under GPL-3.0 (the leading candidate for the app itself).

---

## 🗓️ Beta release

**Target window:** 2026 — exact date confirmed once UI polish + screenshot pass + license decision complete.

**Want to be a beta tester?** ⭐ Star this repository to be notified when the beta announcement lands, or open a GitHub issue to register interest now. Tester slots will fill in the order interest was registered.

What we'll ask of beta testers:
- Try real workflows on real markets (not just dry runs).
- Report bugs against this repo.
- Optionally — share screenshots / short videos for the gallery section above.

What you get:
- Early access before public release.
- Direct line to the maintainer for tracking your reports.
- Credit (with consent) in the v1 release notes.

---

## 📋 Public roadmap

- [x] V3 plugin architecture
- [x] Real-time execution-report push channel
- [x] Marketplace cryptographic signing + CI verification + drift detection
- [x] Trusted-only mode + Developer-Mode toggle
- [x] In-app marketplace installer
- [x] Strategy + indicator consent gate
- [x] Marketplace public flip (Phase M4 — 2026-04-29)
- [x] R3 review cycle (22c + 64M + 26m findings closed)
- [ ] UI polish + screenshot capture pass
- [ ] License terms decision
- [ ] Symbol-split implementation (currently merge-only)
- [ ] Auto-update for the in-app marketplace installer (deferred to v1.1)
- [ ] Beta launch

---

## ⚖️ License

**License terms are currently under consideration.** No license file is shipped in this repository today; the source code is not yet publicly distributed. Once the beta release is approved, the chosen license will be applied retroactively to the published source and announced here.

Until that time, all rights to the OpenTrader Pro codebase are reserved by the maintainer.

The companion `OpenTraderPro-MarketPlace` repository ships its example plugins / strategies / indicators under **GPL-3.0**, which is the leading candidate for the main app's release license — but no commitment yet.

---

## 📧 Contact + updates

- **Repository owner:** [@pparesh25](https://github.com/pparesh25)
- **Updates:** ⭐ Star this repository to be notified of beta announcements
- **Beta interest / questions:** open a GitHub issue here

---

_Last updated: 2026-04-29_
