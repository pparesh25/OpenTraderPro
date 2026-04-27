# OpenTrader Pro

> A desktop-first multi-broker trading + analytics platform for Indian equity, F&O, commodity, and global crypto markets. Built in Python + Qt. Local-first. Plugin-extensible. **Currently in the final phase of development + testing.**

---

## ✨ Highlights

### 📈 Offline classic charting
Candlestick + line + OHLCV charts that **work entirely offline** on locally cached historical data. No internet required for analysis once data is downloaded — keep researching during market downtime, internet outages, or while travelling. Indicator overlays, multi-timeframe synchronisation, drawing tools, and pan/zoom — all native, no browser, no JS.

### 🗄️ Local database — your data stays with you
Every tick, every bar, every order, every fill — stored in a **local SQLite database** under `~/.opentrader-pro/`. No cloud lock-in. No vendor pulling the plug on your history. Export anytime in CSV / JSON / Parquet for your own analysis tools. Survives broker API deprecations.

### 🔁 Backtest the same strategy you'll run live
The strategy framework runs **the same Python code path** against historical data and live market data. What you backtest is what you ship — no separate "backtest engine" with subtle behaviour differences. Walk-forward analysis, slippage modelling, and per-fill fee accounting included.

### 🔌 Multi-broker by design
| Region | Broker | Coverage |
|---|---|---|
| 🇮🇳 India | Zerodha (Kite Connect) | NSE / BSE / NFO / MCX / CDS — equity, F&O, commodity |
| 🌐 Crypto | Binance | Spot + USD-M futures + COIN-M futures |
| 🌍 Global data | OpenBB Platform | Multi-provider equity / forex / fundamental data |

Each broker ships as a **swappable user-installable plugin** via the [companion marketplace repository](#-marketplace) — you only run what you need. Add a new broker without recompiling the app.

### 📊 20+ pre-built indicators, infinitely extensible
SMA, EMA, DEMA, TEMA, WMA, VWAP, Parabolic SAR, Bollinger Bands, Keltner Channels, Ichimoku Cloud, RSI, MACD, Stochastic, ADX, Aroon, ATR, CCI, MFI, OBV, Williams %R — all ship in the marketplace as editable `.txt` Python files. **Author your own indicator in 30 lines of Python** — drop it into `~/.opentrader-pro/indicators/<category>/`, restart, and it appears in the chart panel.

### 📜 Strategy framework with carry-forward, intraday, and EOD square-off
Three reference strategies ship in the marketplace; the framework supports cron-style scheduling, per-segment market-hours awareness, holiday calendar gating, and timezone-correct daily counter rollover.

### 🛡️ Risk + compliance built-in
- **Daily order counter** that rolls on each plugin's local midnight (so an Indian-broker counter rolls at IST midnight and a Binance counter rolls at UTC midnight).
- **Holiday calendar** — plugin-supplied baseline + user override JSON; the signal router refuses to send orders on a closed day.
- **Market-hours gate** with per-segment windows (equity 09:15-15:30 IST, commodity 09:00-23:55 IST, crypto 24/7 UTC).
- **Retry queue with sliding-window rate limits** — Zerodha 10/sec + 3 queries/sec; Binance 50 orders / 10s.
- **Typed broker error taxonomy** — auth / transient / rate-limit / insufficient-funds / invalid-order / market-closed / broker-reject — for surgical retry decisions.
- **SEBI static-IP routing** via SOCKS5 tunnel — order placement traverses a static-IP egress while market-data reads stay direct.
- **Daily OAuth refresh** for daily-token brokers (Zerodha 06:00 IST expiry handled with manual paste flow).

### 💼 Trade journal with full tax breakdown
Every fill recorded with broker-supplied `executionReport` data (V3 §F7 push channel — no polling lag), per-segment fee schedule applied automatically. Indian-broker rows expand to the flat tax shape (STT / GST / SEBI charges / stamp duty) for compliance reporting; crypto rows show maker/taker commission. Pair-matching logic for FIFO / LIFO P&L. CSV export per financial year.

### 🧪 Paper trading mode
Wrap any broker plugin with the `PaperTradingService` and route order placement through an in-memory book while reads still hit live broker context (real margins, real fee schedule, real holiday calendar). Switch back to live with one settings change. Per-account opt-in via `paper_account_aliases` setting.

### 🎯 Quick Order with pre-flight safety checks
One-window order entry with currency-aware price labels, segment-correct product/order-type combos, and a pre-order tunnel-status warning (Cancel-default popup if the SOCKS5 tunnel is down before a routed order goes through).

---

## 📸 Screenshots

> _Screenshots coming soon — UI is being polished for beta release._

```
[ Main window — chart + watchlist + order panel ]
[ Trade journal with tax breakdown ]
[ Strategy panel with backtest results ]
[ Quick Order with pre-flight warnings ]
[ Settings → Routed plugins indicator ]
```

---

## 🏗️ Architecture at a glance

| Layer | Tech |
|---|---|
| UI | PySide6 (Qt 6.10) — native desktop, no Electron, no browser |
| Language | Python 3.13+ |
| Persistence | SQLite (local-only by default) |
| Plugin loading | `compile()` + `exec()` from user-installable `.txt` files |
| Background work | Qt signals + `QueuedConnection` — broker reactor threads never touch UI thread |
| Routing | `socks5h://` (DNS-on-remote) for SEBI compliance; live-resolver pattern degrades gracefully when tunnel drops |
| Schedule | APScheduler with per-plugin timezone awareness |

**Local-first.** No mandatory cloud account. No telemetry. Your broker keys live in your OS keychain (macOS Keychain / Windows Credential Manager / Linux Secret Service). Your trades, orders, fills, and strategy state stay on your disk.

**Plugin-extensible.** The app's `_BUILTIN_PLUGIN_MODULES` ships only `mock` + `backtest` (broker-agnostic sandbox tools). Real brokers, strategies, and indicators are user-installable via the [marketplace repository](#-marketplace).

---

## 🛒 Marketplace

The `OpenTraderPro-MarketPlace` repository hosts the canonical plugins, strategies, and indicators as plain `.txt` files. Users download a file from there and drop it into the matching `~/.opentrader-pro/{plugins,strategies,indicators}/<...>/` folder.

> **Marketplace is currently PRIVATE during development + testing.** Public release will land alongside the beta. Email the maintainer for early-access if you want to evaluate the plugin architecture today.

---

## 🚦 Development status

**This project is in the final phase of development and testing.**

- ✅ Core architecture stabilised (V3 plugin system + R2 broker relocation complete).
- ✅ Multi-broker plugin contract finalised; reference plugins (Zerodha, Binance, OpenBB) shipping via marketplace.
- ✅ Strategy framework + backtesting + 20+ indicators ready for early testers.
- 🚧 Marketplace cryptographic signing infrastructure (planned).
- 🚧 In-app marketplace fetcher (planned).
- 🚧 Strategy + indicator consent gate (security hardening, planned).
- 🚧 UI polish + screenshot pass for marketing.

**Source code is not yet publicly available.** The release process — including the choice of open-source license terms for the beta + general availability — is currently under consideration. Updates will be published here when license decisions are finalised.

---

## 🗓️ Beta release

**Target window:** 2026 — exact date to be confirmed once the remaining hardening phases (marketplace signing, in-app fetcher, consent gate) and UI polish complete.

**Want to be a beta tester?** Watch this repository for the announcement, or open a GitHub issue to register interest.

---

## 📋 Public roadmap

- [x] V3 plugin architecture (5 phases — F1-F9, N1-N4, P1-P5, R1-R4)
- [x] Real-time `executionReport` push channel (V3 §F7)
- [x] Marketplace repository seeded with reference plugins / strategies / indicators
- [ ] Marketplace cryptographic signing (Ed25519) — planned
- [ ] Trusted-only mode + developer-mode toggle in app settings — planned
- [ ] In-app marketplace fetcher (download + verify + cache + auto-update opt-in) — planned
- [ ] Strategy + indicator consent gate — planned (closes the existing security gap that plugins already cover)
- [ ] License terms decision for beta release — under consideration
- [ ] UI polish + documentation pass for marketing
- [ ] Beta launch

---

## ⚖️ License

**License terms are currently under consideration.** No license file is shipped in this repository today; the source code is not yet publicly distributed. Once the beta release is approved, the chosen license will be applied retroactively to the published source and announced here.

Until that time, all rights to the OpenTrader Pro codebase are reserved by the maintainer.

The companion `OpenTraderPro-MarketPlace` repository (currently private) ships its example plugins / strategies / indicators under **GPL-3.0**, which is the leading candidate for the main app's release license — but no commitment yet.

---

## 📧 Contact + updates

- **Repository owner:** [@pparesh25](https://github.com/pparesh25)
- **Updates:** ⭐ Star this repository to be notified of beta announcements
- **Beta interest / questions:** open a GitHub issue here

---

_Last updated: 2026-04-27_
