# RZ AI TRADER — Security & Safety Constraints

These rules are non-negotiable and apply to every phase of the project. They are
enforced by design, not by convention.

## 1. Single authorized user only

- Exactly one `users` row may exist. Registration is never implemented.
- No email collection, no OAuth, no invite links, no "create account" page.
- Initial setup creates the admin user once (password hash stored in SQLite).
- All API routes require a valid session. Absence of a valid session → 401.

## 2. No public registration

- `/auth/register` or equivalent must never exist.
- The frontend has no sign-up flow.
- The admin user is created through a one-time setup step (CLI or setup route
  that disables itself after first use).

## 3. No real-money trading

- The codebase never calls any Binance endpoint that places, cancels, or modifies
  an order (no `/api/v3/order`, `/fapi/v1/order`, etc.).
- No connection to any exchange trading API.
- The platform is read-only with respect to the exchange: market data in,
  nothing out.
- Paper trading is simulated entirely in the local SQLite DB; no exchange
  interaction.

## 4. No automatic order execution

- Signal engine outputs advisory signals (`Long / Short / Neutral`) only.
- There is no strategy executor, no auto-trader, no bot loop that sends orders.
- All trade decisions remain manual; the user reads signals and decides.

## 5. Never request, store, or transmit API secrets

- The app does not ask the user for a Binance API key, API secret, or passphrase.
- `.env` and the database contain no `BINANCE_API_KEY`, `BINANCE_SECRET`,
  `apiKey`, or `secret` fields.
- `ccxt` is configured for public endpoints only (no `apiKey`/`secret` set).
- Binance public market-data endpoints (`/api/v3/...`, `/fapi/v1/...`, public WS)
  require no credentials and are the only Binance endpoints used.

## 6. Free tools and open-source libraries only

- Backend: Python + FastAPI + SQLAlchemy + pandas-ta + ccxt (all MIT/BSD/PSF).
- Frontend: React + Vite + Tailwind + Lightweight Charts (all MIT).
- No paid AI models, no paid APIs, no paid market data, no paid infrastructure.
- No credits are spent.

## 7. Network boundaries

- Outbound network access limited to Binance public endpoints:
  - `api.binance.com` (spot REST)
  - `fapi.binance.com` (futures REST)
  - `stream.binance.com` (spot WS)
  - `fstream.binance.com` (futures WS)
- No outbound to model providers, analytics services, or third-party APIs.
- Inbound: localhost-only by default (`127.0.0.1`). Do not bind to `0.0.0.0` or
  expose to the public internet.
- No webhooks, no outbound order payloads, no email/SMS providers.

## 8. Data handling

- All data persists locally in a single SQLite file (`backend/data/app.sqlite`).
- No cloud database, no third-party logging of user data, no telemetry.
- Back up the SQLite file manually; the platform does not export user data to
  external services.

## 9. Enforcement (how we verify)

A security grep must return clean results before any release:

```
grep -rnE "apiKey|secret|apisecret|place_order|create_order|cancel_order|order_id" \
  backend/app frontend/src --exclude-dir=node_modules --exclude-dir=__pycache__
```

- Any hit in application source (not docs/tests) is a blocker.
- `.env` must be in `.gitignore`; no real values committed.
- `docker-compose.yml` production profile binds backend to `127.0.0.1` only.

## 10. Out of scope (explicitly not built)

- Exchange API key/secret UI.
- Real order placement, cancellation, or modification.
- Auto-trading bots or scheduled execution.
- Multi-user accounts, teams, or registration.
- Paid market data feeds or paid AI models.
- Public internet exposure.
