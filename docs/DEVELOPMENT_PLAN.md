# RZ AI TRADER — Development Plan

## How to read this plan

The work is split into 11 phases. Each phase lists:
- **Goal** — what the phase achieves.
- **Deliverables** — concrete files/features produced.
- **Acceptance criteria** — how to verify the phase is done.
- **Dependencies** — prior phases that must be complete.

**Rule:** No implementation begins until this plan and the architecture doc are reviewed. No real-money, no API secrets, no auto-execution at any phase.

---

## Phase 0 — Project foundation (Week 1)

Goal: Set up the repository skeleton, backend, frontend, database, and single-user auth so later phases have a working foundation.

Deliverables:
- `backend/` FastAPI app with `main.py`, SQLAlchemy + SQLite, Alembic migrations.
- `frontend/` React + Vite + Tailwind app, dev server on localhost.
- `docker-compose.yml` + `Dockerfile` for local one-command startup.
- `users` table with a single setup-created row; bcrypt password hash; session cookie (HttpOnly, SameSite=Strict).
- Login page + logout; middleware requiring session on all API routes.
- `.env.example` with `ADMIN_PASSWORD_HASH`, `SESSION_SECRET`, `BINANCE_BASE_URL`. No API key/secret fields.
- Health check endpoint `GET /health`.

Acceptance criteria:
- `docker compose up` starts backend + frontend; `GET /health` returns 200.
- Login with the setup password sets a session cookie; protected routes return 401 without it.
- No registration endpoint exists.

Dependencies: None.

---

## Phase 1 — Binance public market data (Week 1–2)

Goal: Connect to Binance public REST/WebSocket market data and persist candles. No credentials required.

Deliverables:
- `backend/app/services/binance_client.py` — async client for public endpoints:
  - `GET /api/v3/klines` (candles), `GET /api/v3/ticker/price`, `GET /api/v3/depth` (order book snapshot).
  - Futures: `GET /fapi/v1/ticker/openInterest`, `GET /fapi/v1/fundingRate`.
- WebSocket subscriptions: `kline_1m`, `bookTicker`, `aggTrade` for watched symbols.
- `models/` — `Candle`, `Ticker`, `OrderBookSnapshot`, `OpenInterest`, `FundingRate` ORM models.
- `services/market_ingest.py` — background task that polls REST + maintains WS, writes candles to SQLite.
- API routes: `GET /api/market/candles/{symbol}`, `GET /api/market/ticker/{symbol}`, `GET /api/market/orderbook/{symbol}`.
- Frontend: watchlist component showing live prices for user-selected symbols.

Acceptance criteria:
- Real-time candle updates arrive for BTC/USDT without any API key.
- Order book snapshot returns bids/asks; open interest and funding rate return values.
- Frontend watchlist updates prices without page reload.

Dependencies: Phase 0.

---

## Phase 2 — Candlestick charts & technical analysis (Week 2–3)

Goal: Render financial charts and compute indicators locally.

Deliverables:
- `backend/app/services/ta_engine.py` — `pandas-ta`-based indicators: EMA, SMA, RSI, MACD, Bollinger Bands, ATR, VWAP, Supertrend.
- API routes: `GET /api/analysis/indicators/{symbol}` returning indicator values for the latest candle.
- `frontend/src/charts/` — Lightweight Charts (TradingView) wrapper: candlestick series + indicator overlays (EMA, Bollinger) and pane studies (RSI, MACD).
- Frontend chart page: symbol selector, timeframe selector (1m, 5m, 15m, 1h, 4h, 1d), drawing of candles + indicators.

Acceptance criteria:
- Chart displays candles from persisted SQLite data.
- At least 5 indicators overlay correctly on the chart.
- Indicator values computed match `pandas-ta` output (unit-tested).

Dependencies: Phase 1.

---

## Phase 3 — Support & resistance detection (Week 3)

Goal: Detect S/R zones from price action and show them on the chart.

Deliverables:
- `backend/app/services/sr_detector.py` — swing-high/swing-low pivot detection; zone merging; strength scoring by touch count.
- `models/` — `SupportResistanceZone` ORM model.
- API routes: `GET /api/analysis/sr/{symbol}` returning zones with price levels, strength, and type (support/resistance).
- Frontend: horizontal zone bands drawn on the chart (light shaded rectangles + labels).

Acceptance criteria:
- Zones are computed from persisted candles, not hardcoded.
- Zones appear on the chart aligned to price levels.
- Zone strength increases with repeated touches (unit-tested).

Dependencies: Phase 1 (candles), Phase 2 (chart rendering).

---

## Phase 4 — Liquidity & order book analysis (Week 3–4)

Goal: Analyze Binance order book depth for imbalance and walls.

Deliverables:
- `backend/app/services/ob_analyzer.py` — bid/ask imbalance ratio, wall detection (large single orders), spread, liquidity density by price bucket.
- Persisted `OrderBookSnapshot` refreshed via WebSocket `bookTicker` + periodic REST depth.
- API route: `GET /api/analysis/orderbook/{symbol}` returning imbalance %, walls, spread, top bids/asks.
- Frontend: order-book panel (bid/ask ladder or depth chart) with imbalance and wall highlights.

Acceptance criteria:
- Order book data reflects Binance live depth.
- Imbalance % and walls are computed and displayed.
- Panel updates as the book changes.

Dependencies: Phase 1.

---

## Phase 5 — Open Interest & Funding Rate (Week 4)

Goal: Show futures positioning and cost-of-carry signals.

Deliverables:
- `backend/app/services/futures_monitor.py` — polls open interest and funding rate; stores history.
- `models/` — `OpenInterest`, `FundingRate` with timestamps.
- API routes: `GET /api/futures/open-interest/{symbol}`, `GET /api/futures/funding/{symbol}`.
- Frontend: funding-rate gauge and OI trend mini-chart on the dashboard/symbol page.

Acceptance criteria:
- OI and funding values come from Binance futures public endpoints.
- History is stored and trend is visible.

Dependencies: Phase 1.

---

## Phase 6 — Long / Short signal analysis (Week 4–5)

Goal: Combine all inputs into a directional signal with confidence.

Deliverables:
- `backend/app/services/signal_engine.py` — rule-based signal composition:
  - TA score (RSI overbought/oversold, MACD cross, EMA alignment, Supertrend).
  - S/R proximity (price near support → long bias; near resistance → short bias).
  - Order book imbalance (strong bid side → long bias).
  - Funding rate (extreme positive → contrarian short bias; negative → long bias).
  - Open interest trend (rising OI + price up = long conviction).
- Output: `Long | Short | Neutral` + confidence score (0–100) + contributing factors.
- `models/` — `Signal` ORM model (symbol, direction, confidence, factors, timestamp).
- API routes: `GET /api/signals/{symbol}`, `GET /api/signals` (all active).
- Frontend: signal panel per symbol showing direction, confidence, and factor breakdown.

Acceptance criteria:
- Signal is a deterministic function of inputs (unit-tested with fixed inputs).
- All five input categories contribute to the confidence score.
- No signal ever triggers an order; display only.

Dependencies: Phases 2–5.

---

## Phase 7 — Risk management (Week 5)

Goal: Provide read-only risk guidance for manual trades.

Deliverables:
- `backend/app/services/risk_engine.py` — position-size calculator:
  - Risk per trade = fixed % of account equity (user-configurable, default 1%).
  - Stop-loss from ATR or swing low/high.
  - Position size = risk equity / (entry − stop).
  - Risk/reward ratio for take-profit targets.
  - Max daily loss guard (informational only).
- `models/` — `RiskSettings` (single row, user-editable).
- API routes: `GET /api/risk/calculate`, `GET/PUT /api/risk/settings`.
- Frontend: risk calculator form (entry price, stop, equity) outputting position size, R/R, and risk amount.

Acceptance criteria:
- Calculations are mathematically correct (unit-tested).
- Outputs are advisory only; no order is placed.
- Settings persist to SQLite.

Dependencies: Phase 1 (ATL/ATR data), Phase 6 (signal for entry context optional).

---

## Phase 8 — Backtesting & paper trading (Week 6–7)

Goal: Simulate strategy performance and paper-trade without real money.

Deliverables:
- `backend/app/services/backtest_engine.py` — replays historical candles through `signal_engine`; reports win rate, expectancy, max drawdown, profit factor, trade list.
- `backend/app/services/paper_engine.py` — simulated order ledger: open/close paper positions at live/historical prices, track PnL, no exchange calls.
- `models/` — `BacktestRun`, `BacktestTrade`, `PaperPosition`, `PaperTrade`.
- API routes: `POST /api/backtest/run`, `GET /api/backtest/runs`, `GET /api/paper/positions`, `POST /api/paper/orders` (simulated only), `GET /api/paper/trades`.
- Frontend: backtest results page (metrics + equity curve chart); paper trading page (positions, PnL, simulated order entry).

Acceptance criteria:
- Backtest runs on stored historical data and produces correct metrics (verified against a manual calc on a small sample).
- Paper positions reflect simulated fills at real prices; no Binance trading endpoint is called.
- Paper PnL updates as market price moves.

Dependencies: Phases 1, 6.

---

## Phase 9 — Private dashboard (Week 7–8)

Goal: Single private view aggregating everything.

Deliverables:
- `frontend/src/pages/Dashboard.tsx` — watchlist with live prices, active signals, chart preview, risk readout, paper PnL summary.
- Navigation shell: Dashboard, Chart, Analysis, Backtest, Paper, Settings.
- Settings page: change password, risk % default, watched symbols, timeframes.
- Responsive layout, dark theme suitable for a trading terminal.

Acceptance criteria:
- All major data points visible on one page without auth bypass.
- Settings changes persist.
- No registration or multi-user UI anywhere.

Dependencies: Phases 0–8.

---

## Phase 10 — Hardening, testing & docs (Week 8–9)

Goal: Harden security, add tests, write documentation.

Deliverables:
- Backend `pytest` suite covering: auth middleware, signal engine determinism, risk calcs, backtest metrics, SR detection.
- Frontend unit tests for chart and signal components.
- Input validation on all API routes (Pydantic); rate-limit public endpoints lightly.
- Security review: confirm no API key/secret fields anywhere; confirm `.env` has no secrets committed; confirm Binance trading endpoints are never called.
- `README.md` with setup, run, and security policy.
- `.gitignore` excluding `.env`, `*.sqlite`, `node_modules`, `__pycache__`.

Acceptance criteria:
- All tests pass.
- `grep` for `apiKey`, `secret`, `sign`, `place_order`, `create_order` returns only documentation stating these are forbidden.
- Build succeeds with `docker compose up --build`.

Dependencies: All prior phases.

---

## Phase 11 — Release prep (Week 9)

Goal: Produce a deployable, single-user artifact.

Deliverables:
- Production `docker-compose.yml` binding backend to `127.0.0.1` only.
- One-command setup script that creates the admin user and hashes the password.
- Deployment notes: run locally, do not expose to public internet, back up the SQLite file.

Acceptance criteria:
- Fresh clone → `docker compose up` → working private dashboard.
- No exposed ports beyond localhost by default.

Dependencies: Phase 10.

---

## Timeline summary

| Phase | Focus | Est. |
|-------|-------|------|
| 0 | Foundation + auth | Week 1 |
| 1 | Binance public market data | Week 1–2 |
| 2 | Charts + technical analysis | Week 2–3 |
| 3 | Support & resistance | Week 3 |
| 4 | Order book / liquidity | Week 3–4 |
| 5 | Open interest & funding | Week 4 |
| 6 | Long/Short signals | Week 4–5 |
| 7 | Risk management | Week 5 |
| 8 | Backtesting + paper trading | Week 6–7 |
| 9 | Private dashboard | Week 7–8 |
| 10 | Testing + hardening | Week 8–9 |
| 11 | Release prep | Week 9 |

Total: ~9 weeks, single developer. Phases 1–6 can overlap where independent.

---

## Definition of "done" for the whole platform

- All 11 phases complete and acceptance criteria met.
- Tests pass; security review confirms no API secrets and no trading-endpoint calls.
- The app runs locally as a single-user, private dashboard using only free OSS tools and public Binance market data.
- No real money is touched at any point.
