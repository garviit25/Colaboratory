# CLAUDE.md — Fyers Trading Dashboard

> Place this file in your repo root. Claude Code reads it automatically on every session.

## What This Project Is

A professional trading dashboard for the Indian equity & derivatives market, integrated with **Fyers API v3**. React frontend with a drag-and-resize widget workspace. Python (FastAPI) backend. PostgreSQL + Redis for data.

## Architecture

```
Fyers WS/REST
      │
      ▼
┌─────────────┐    Redis pub/sub     ┌─────────────┐     WebSocket      ┌──────────┐
│ marketdata  │─────────────────────▶│  gateway    │◀──────────────────▶│  React   │
│ (1 socket)  │  tick/depth/trade    │  (FastAPI)  │                    │  Web App │
└─────────────┘                      └─────────────┘                    └──────────┘
                                           │
                    ┌──────────────────┬────┴─────┬──────────────┐
                    ▼                  ▼          ▼              ▼
              ┌──────────┐     ┌──────────┐ ┌─────────┐  ┌──────────┐
              │ spreads  │     │   oms    │ │  risk   │  │  alerts  │
              │ engine   │     │ (Fyers)  │ │ engine  │  │  worker  │
              └──────────┘     └──────────┘ └─────────┘  └──────────┘
```

**Golden rules:**
1. Only `marketdata` service opens Fyers WebSocket connections. Everything else reads Redis.
2. Synthetic spread symbols (`SYN:*`) are treated identically to real symbols everywhere.
3. Frontend never sees raw Fyers symbol strings — the gateway translates.
4. All secrets via `.env` + `pydantic-settings`. Never log tokens. Pre-commit `gitleaks` hook.
5. All prices formatted via `formatPrice(symbol, price)` using tick size from contract master.

## Tech Stack

- **Frontend:** React 18 + TypeScript + Vite, `react-grid-layout` (widget workspace), `zustand` (state), TradingView Lightweight Charts, Tailwind CSS
- **Backend:** Python 3.11, FastAPI, `uvicorn`, `fyers-apiv3`
- **Data:** PostgreSQL (TimescaleDB for ticks), Redis (pub/sub + quote cache)
- **Testing:** pytest, Playwright, property tests with `hypothesis`

## Folder Structure

```
/apps/web/                    React dashboard
/services/
  auth/                       Fyers OAuth + token refresh
  marketdata/                 WebSocket fan-out → Redis
  contracts/                  Symbol master, lot sizes, expiries
  spreads/                    Spread pricing engine
  oms/                        Order management (Fyers REST)
  risk/                       Exposure calculator
  alerts/                     Rule engine + notifications
/packages/
  shared-types/               Pydantic models → generated TS types
/tests/
  integration/
  e2e/
/docs/
```

## Feature Specifications

### Feature 1: Spread Prices

Continuously compute and publish three spreads for every underlying that has F&O:
- `SPOT - CURRENT_MONTH_FUTURE`
- `SPOT - NEXT_MONTH_FUTURE`
- `CURRENT_MONTH_FUTURE - NEXT_MONTH_FUTURE`

Implementation:
- On startup, the spreads service queries the contract master for all underlyings with active futures.
- For each pair, subscribe to both legs via Redis `tick:{symbol}`.
- On any tick update, recompute: `spread_price = leg1_mid - leg2_mid` (use bid/ask midpoint, NOT LTP).
- Publish the result as `SYN:SPREAD:{underlying}:{type}` on the same Redis bus.
- These synthetic symbols feed into ladders and charts like real symbols.

A spread **ladder** shows combined price rungs. A spread **chart** shows historical spread values (store computed ticks in TimescaleDB `synthetic_ticks` hypertable).

### Feature 2: Custom Spread Maker

UI modal to construct any combination of up to 4 futures contracts:
- Each leg: symbol picker (searchable), buy/sell toggle, **price multiplier** (float input, default 1.0).
- Formula: `Σ (side_i × multiplier_i × price_i)` where `side` is +1 for buy, -1 for sell.
- Named and saved to Postgres `custom_spreads` table.
- Published as `SYN:CUSTOM:{name}` on Redis, usable in any widget.
- Must have a chart and ladder view for each custom spread.
- Multiplier changes recompute the spread in real time.

### Feature 3: Ladder Trading

Reference: TT-style market ladder (see attached reference image).

Columns: `Working Orders | Bids | Price | Asks | Volume-at-Price`

Requirements:
- Minimum 5 bid and 5 ask levels visible (configurable, default 10 each side).
- Price rungs auto-centered on LTP. Manual re-center button.
- Color coding: blue for bids, red for asks, green highlight on LTP, yellow flash on last-traded rung.
- Click on any bid/ask rung to place a limit order at that price.
- Order quantity set via quick-select buttons (1, 5, 10, 50, 100, custom) on the widget.
- Working orders displayed in the leftmost column with cancel-on-click.
- CXL S (cancel all sells), CXL B (cancel all buys), CXL (cancel all) buttons.
- Orders placed via Fyers REST `place_order` endpoint through the OMS service.
- Ladder must work for both real symbols AND synthetic spread symbols.

### Feature 4: Risk Analysis Page

Separate full-page view (not a widget — a route `/risk`).

**Exposure calculation rules (exactly as specified):**

| Position Type | Exposure Formula | Direction |
|---|---|---|
| Futures Long | `lot_size × LTP × num_lots` | Long |
| Futures Short | `lot_size × LTP × num_lots` | Short |
| Option Sell (Call) | `strike_price × lot_size × num_lots` | Short |
| Option Sell (Put) | `strike_price × lot_size × num_lots` | Long |
| Option Buy (Call) | `lot_size × option_LTP × num_lots` | Long |
| Option Buy (Put) | `lot_size × option_LTP × num_lots` | Short |

Display:
- Group by underlying.
- Show: gross long exposure, gross short exposure, net exposure per underlying.
- Portfolio totals at the bottom.
- All values update live as LTP changes.

### Feature 5: Hypothetical Position Builder

On the Risk Analysis page, a section labeled "What-If Analysis":
- "Add Hypothetical Position" button opens a form: instrument picker, buy/sell, quantity, price (defaults to LTP).
- Hypothetical positions appear in the same exposure table but visually distinct (dashed border, "HYPOTHETICAL" badge, different background color).
- Exposure recalculates live including hypotheticals.
- "Clear All Hypotheticals" button.
- "Promote to Real" button on each hypothetical row → opens the ladder pre-filled with that order.
- Hypotheticals are stored in browser `zustand` state (not persisted to server).

### Feature 6: Alerts

Alert rule engine supporting:
- **Instruments:** any real symbol (futures, options, equity) AND synthetic symbols (spreads, custom strategies).
- **Conditions:** price `>`, `<`, `crosses above`, `crosses below`, `between` a range.
- **T&S alerts:** "trade size > X lots within Y seconds" on any symbol.
- **Custom spread alerts:** alert when a custom spread price hits a threshold.

Implementation:
- Alerts evaluated in a single async worker consuming from Redis streams (not polling).
- Dispatch: in-app toast notification + sound. Optional webhook URL per alert.
- Alert CRUD UI: create, edit, delete, enable/disable. Show alert-fired history.
- Persist rules in Postgres `alert_rules` table.

### Feature 7: Time & Sales

Reference: see attached T&S screenshot.

Columns: `Time | Price | Qty`

- Color-coded rows: red = trade at/below bid (seller-initiated), green = trade at/above ask (buyer-initiated), neutral = mid.
- Auto-scrolling with pause-on-hover.
- Configurable minimum quantity filter (hide small trades).
- Each row shows the exact timestamp (HH:MM:SS format).
- T&S data sourced from Fyers WebSocket trade channel, published to Redis `trade:{symbol}`.
- **T&S alerts** (from Feature 6) integrate here: highlight rows that triggered an alert.

## Widget Workspace

The main dashboard (`/`) is a `react-grid-layout` workspace where users can:
- Add widgets: Ladder, Chart, Time & Sales (via an "Add Widget" toolbar).
- Each widget instance is independently configurable (its own symbol).
- Drag to reposition, resize handles on edges.
- Layout persists to `localStorage`.
- Multiple saved layouts (dropdown: "Scalping", "Spreads", "Monitoring") stored in `localStorage`.

## Simulation Data for Testing

Since Fyers doesn't have a sandbox, build a **tick replay mock**:
- `/services/marketdata/mock.py`: reads recorded tick data from a JSON file and publishes to Redis at configurable speed.
- Record format: `[{timestamp, symbol, ltp, bid, ask, bid_qty, ask_qty, volume, trade_qty}, ...]`
- Include a seed script that generates realistic sample data for: NIFTY spot + 2 futures + 10 option strikes, RELIANCE spot + 2 futures.
- All integration tests and Playwright E2E tests run against this mock, never against live Fyers.
- The mock must also simulate L2 depth (5 levels) and trade-by-trade data for T&S.

## Fyers API Reference

- **Auth:** OAuth2 flow → `POST /api/v3/token` → access_token. Short-lived; refresh proactively.
- **Symbol master:** `GET /api/v3/symbols` — download CSV, parse into contract master daily.
- **Quotes:** WebSocket `wss://socket.fyers.in/hsm_stream` — modes: LTP, Quote (bid/ask), Full (L2 depth).
- **Orders:** `POST /api/v3/orders` (place), `PUT /api/v3/orders` (modify), `DELETE /api/v3/orders/{id}` (cancel).
- **Positions:** `GET /api/v3/positions` — current open positions.
- **Depth:** Available through WebSocket in Full mode — 5 levels of bid/ask.

**Gotchas:**
- WebSocket has a per-socket symbol subscription limit. Shard across sockets if needed; all publish to same Redis channels.
- Reconnects lose subscriptions silently. Re-subscribe on every reconnect.
- Option symbols change format near expiry. Never regex-parse — join against contract master.

## Development Commands

```bash
# Backend
cd services && python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"
docker compose up -d postgres redis
uvicorn gateway.main:app --reload --port 8000

# Frontend
cd apps/web && pnpm install && pnpm dev

# Test
pytest -x                    # backend
pnpm test                    # frontend
pnpm test:e2e                # Playwright

# Mock data mode (no Fyers account needed)
MOCK_MODE=true uvicorn gateway.main:app --reload
```

## Build Order (follow this sequence)

1. Scaffold monorepo, docker-compose, CI
2. Fyers auth flow + token management
3. Contract master ingest + search API
4. WebSocket fan-out + Redis pub/sub + mock replay
5. Widget workspace (empty grid with add/remove/resize/persist)
6. Ladder widget (real symbols first)
7. Chart widget
8. Time & Sales widget
9. OMS integration (order placement from ladder)
10. Spread pricing engine + synthetic symbols
11. Custom spread builder UI
12. Ladder + Chart for synthetic symbols
13. Risk analysis page + exposure calc
14. Hypothetical position builder
15. Alert rule engine + alert CRUD UI
16. T&S alert integration
17. Polish: hotkeys, themes, layout presets

**Do NOT skip ahead.** Each step depends on the previous ones working. Test each step against the mock before moving on.

## Code Style

- Python: `ruff` lint + format, `mypy --strict`
- TypeScript: ESLint + Prettier, strict mode
- Commits: conventional commits (`feat(ladder):`, `fix(spreads):`)
- All time internally UTC, convert to Asia/Kolkata only at UI boundary
- Never `.toFixed(2)` — always use `formatPrice()` with tick size from contract master

## Files to Ignore

```
# .claude/settings.json
{
  "ignoreFiles": [
    "node_modules/",
    "dist/",
    ".venv/",
    "__pycache__/",
    "*.pyc",
    "test_data/recordings/"
  ]
}
```
