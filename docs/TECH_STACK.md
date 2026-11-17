# RZ AI TRADER — Tech Stack

## Constraint summary

- Free and open-source only. No paid models, no paid APIs, no credits.
- Single-user, local-first. No registration.
- Public Binance market data only. No credentials.

## Backend

| Component | Choice | Version (target) | License | Purpose |
|-----------|--------|------------------|---------|---------|
| Language | Python | 3.11+ | PSF | Application logic |
| Framework | FastAPI | 0.100+ | MIT | Web API, async |
| ASGI server | Uvicorn | latest | BSD | Serve the API |
| ORM / migrations | SQLAlchemy + Alembic | 2.0 / latest | MIT | SQLite DB + schema versioning |
| Database | SQLite | bundled | public domain | Single-user file DB |
| Validation | Pydantic | v2 | MIT | Request/response schemas |
| Password hashing | passlib + bcrypt | latest | BSD / MIT | Single-user password |
| Session | itsdangerous / itsdangerous-signed cookie or fastapi-sessions | latest | MIT | Server-side session cookie |
| Binance client | ccxt + websockets | latest | MIT | Public market data (no key) |
| Technical analysis | pandas-ta | latest | MIT | Indicators on pandas |
| HTTP client | httpx | latest | BSD | Binance REST calls |
| Background tasks | FastAPI BackgroundTasks / asyncio | — | — | Market data ingestion loop |
| Testing | pytest + pytest-asyncio | latest | MIT | Backend unit/integration tests |
| Lint/format | ruff | latest | MIT | Lint + format |

## Frontend

| Component | Choice | Version (target) | License | Purpose |
|-----------|--------|------------------|---------|---------|
| Language | TypeScript | 5.x | Apache 2.0 | Type safety |
| Framework | React | 18 | MIT | UI |
| Build tool | Vite | 5.x | MIT | Dev server + build |
| Styling | Tailwind CSS | 3.x | MIT | Utility-first CSS |
| Charts | Lightweight Charts (TradingView) | latest | MIT | Candlesticks + studies |
| State | Zustand (or React Context) | latest | MIT | Client state |
| HTTP client | fetch (native wrapper) or axios | — | — | API calls |
| Testing | Vitest + React Testing Library | latest | MIT | Frontend unit tests |
| Lint/format | ESLint + Prettier | latest | MIT | Lint + format |

## Data / external services

| Source | Endpoint | Auth | Cost |
|--------|----------|------|------|
| Binance Spot | `api.binance.com/api/v3/...` | None (public) | Free |
| Binance Futures | `fapi.binance.com/fapi/v1/...` | None (public) | Free |
| Binance WS Spot | `stream.binance.com:9443/ws/...` | None | Free |
| Binance WS Futures | `fstream.binance.com/ws/...` | None | Free |

No model providers, no paid data feeds, no exchange trading endpoints.

## Optional (dev only)

| Component | Choice | Purpose |
|-----------|--------|---------|
| Container | Docker + docker-compose | One-command local dev |
| Editor config | `.editorconfig` | Consistent formatting |

## Files to create

```
backend/requirements.txt     # fastapi, uvicorn, sqlalchemy, alembic, pydantic,
                              # passlib[bcrypt], ccxt, websockets, httpx,
                              # pandas-ta, pytest, pytest-asyncio, ruff
frontend/package.json        # react, react-dom, typescript, vite, tailwindcss,
                              # lightweight-charts, zustand, vitest, @testing-library
docker-compose.yml           # backend + frontend services
Dockerfile                   # python app image
.env.example                 # template (no real values)
```
