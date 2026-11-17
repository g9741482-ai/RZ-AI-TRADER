# RZ AI TRADER — Project Architecture

## 1. Overview

RZ AI TRADER is a private, single-user AI cryptocurrency futures analysis platform.
It is a read-only analysis tool: it consumes **public** Binance market data, runs
technical analysis locally, and presents signals on a private dashboard.

**Hard constraints (never violated):**
- One authorized user only. No public registration, no sign-up page.
- No real-money trading. No connection to any exchange trading endpoint.
- No automatic order execution. All trade ideas are manual/signal-only.
- Never request, store, or transmit API secrets (API key or secret key). Public
  market data endpoints require no credentials.
- Free and open-source tools only. No paid models, no paid APIs.
- No credits spent.

## 2. System Diagram (conceptual)

```
┌─────────────┐      HTTPS / WSS      ┌─────────────────────┐
│  Browser    │ ◄────────────────────► │  FastAPI Backend    │
│  (React)    │      (same origin)    │  (Python, uvicorn)  │
└─────────────┘                       └─────────┬───────────┘
                                                │ public REST/WSS
                                                ▼
                                       ┌─────────────────┐
                                       │  Binance Public │
                                       │  Market Data    │
                                       └─────────────────┘

Local state (single user, file-based):
  backend/data/  sqlite DB  (candles, indicators, signals, paper trades, backtests)
  backend/config/.env  (single user password hash only — NO API secrets)
```

## 3. Tech Stack

| Layer | Choice | Rationale |
|-------|--------|-----------|
| Language | Python 3.11+ (backend), TypeScript/React (frontend) | Free, OSS, strong data ecosystem |
| Backend framework | FastAPI | Fast, async-native, automatic OpenAPI docs |
| API client to Binance | `ccxt` + `websockets` | Unified OSS library; public endpoints need no key |
| Real-time data | Binance public WebSocket (ticker, bookTicker, aggTrade, kline) | Free, no auth |
| Candlestick charts | Lightweight Charts (TradingView, OSS MIT) | Purpose-built financial charts, free |
| Technical analysis | `pandas-ta` | Open-source TA library built on pandas |
| Database | SQLite via SQLAlchemy + Alembic | Zero-config, single-user file DB, free |
| Auth / session | bcrypt password hash + server-side session cookie | Single user, no registration, no secrets stored |
| Styling | Tailwind CSS | Free, utility-first |
| Build tooling | Vite (frontend), pip + venv (backend) | Free |
| Testing | pytest (backend), Vitest/React Testing Library (frontend) | Free |
| Container (optional) | Docker + docker-compose | Free, local dev only |

## 4. Directory Layout

```
rz-ai-trader/
├── backend/
│   ├── app/
│   │   ├── api/           # route modules (market, chart, analysis, signals, backtest, paper, auth)
│   │   ├── core/          # config, security, logging
│   │   ├── crud/          # DB read/write helpers
│   │   ├── models/        # SQLAlchemy ORM models
│   │   ├── schemas/       # Pydantic request/response schemas
│   │   ├── services/      # binance_client, ta_engine, sr_detector, ob_analyzer, signal_engine, risk_engine, backtest_engine, paper_engine
│   │   └── main.py        # FastAPI app factory
│   ├── alembic/           # DB migrations
│   ├── alembic.ini
│   ├── requirements.txt
│   └── tests/
├── frontend/
│   ├── src/
│   │   ├── components/    # reusable UI (Chart, OrderBook, SignalPanel, ...)
│   │   ├── pages/         # Dashboard, Chart, Analysis, Backtest, Paper, Settings
│   │   ├── services/      # API client (fetch wrapper)
│   │   ├── charts/        # Lightweight Charts wrappers
│   │   ├── store/         # state (Zustand or React context)
│   │   └── main.tsx
│   ├── index.html
│   ├── package.json
│   └── vite.config.ts
├── docker-compose.yml
├── Dockerfile
├── README.md
└── docs/
    ├── PROJECT_ARCHITECTURE.md
    ├── DEVELOPMENT_PLAN.md
    ├── TECH_STACK.md
    └── SECURITY_CONSTRAINTS.md
```

## 5. Data Flow

1. **Market data ingestion** — Backend subscribes to Binance public WebSocket streams
   (`kline_1m`, `bookTicker`, `aggTrade`) for the user's watched symbols. Candles are
   persisted to SQLite. Latest ticker/order-book snapshots are held in memory and
   pushed to the frontend over HTTP polling or a lightweight WS.
2. **Technical analysis** — On request (or periodically), `pandas-ta` computes
   indicators (EMA, RSI, MACD, Bollinger Bands, ATR, VWAP, etc.) from stored candles.
3. **Support/resistance** — Swing-high/low pivot detection on recent candles; zones
   merged and stored as signal metadata.
4. **Order book / liquidity** — Binance `bookTicker` + depth snapshot (public REST
   endpoint `/api/v3/depth`) analyzed for bid/ask imbalance and walls.
5. **Open Interest / Funding** — Public futures endpoints (`/fapi/v1/ticker/openInterest`,
   `/fapi/v1/fundingRate`) polled periodically.
6. **Signal engine** — Combines TA + S/R + OI/funding + order-book imbalance into
   Long/Short/Neutral signal with a confidence score. Display only — no execution.
7. **Risk management** — Position-size calculator (fixed % of equity, stop-loss from
   ATR/swing), risk-reward ratio, max daily loss. Read-only guidance.
8. **Backtesting** — Replays historical candles through the signal engine; reports
   win rate, expectancy, drawdown. Fully simulated.
9. **Paper trading** — Simulated order ledger in SQLite using historical/live prices;
   no exchange interaction.
10. **Dashboard** — Aggregates watchlist, live prices, active signals, chart, and risk
    readout in one private view.

## 6. Authentication Model (single user)

- One `users` row is created during initial setup (one-time).
- Password hashed with bcrypt; stored in SQLite. No email, no OAuth, no registration
- endpoint.
- Session cookie (HttpOnly, SameSite=Strict, secure in production) issued after login.
- Every API route requires a valid session. Unauthorized → 401.
- The `.env` holds only `ADMIN_PASSWORD_HASH` and `SESSION_SECRET`. It never holds
  Binance API keys/secrets.

## 7. Network Boundaries

- Outbound: Binance public REST/WebSocket only (`api.binance.com`, `fapi.binance.com`,
  `stream.binance.com`, `fstream.binance.com`). No outbound to paid APIs or model providers.
- Inbound: localhost only during dev; bind to 127.0.0.1 by default so the dashboard is
  not exposed to the network unintentionally.
- No webhooks, no outbound order payloads, no exchange trading endpoints called.

## 8. Non-Goals (explicitly out of scope)

- Exchange API key/secret input or storage.
- Placing, canceling, or modifying real orders.
- Auto-trading bots or strategy execution.
- Multi-user accounts, registration, or public access.
- Paid market data, paid AI models, or paid infrastructure.
