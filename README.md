# Project Oracle: Real-Time HFT Scalping & Backtesting Architecture Blueprint

## 1. Technology Stack

| Layer | Technology | Purpose |
|---|---|---|
| **Backend Engine** | Python 3.12 + FastAPI | Async API, WebSocket broadcasting, brain mode orchestration |
| **Brain: Pandas Scalp** | `pandas` + `pandas_ta` | HFT scalp scoring engine — tick-level order book + price action |
| **Brain: Claude AI** | Anthropic API (Haiku 4.5) | LLM-based signal generation (future — Intraday mode only) |
| **Brain: Random** | Python `random` module | Dev-only signal generator for UI/DB testing |
| **Legacy Engine** | `scoring.py` (preserved) | Original lagging-indicator engine (ADX, RSI, VADER) — kept for Intraday/Claude mode |
| **Frontend UI** | Angular 19 + `angular-split` | 3-panel resizable real-time dashboard |
| **Charts** | TradingView `lightweight-charts` | Zero-latency candlestick, volume, indicator overlays |
| **Dynamic Indicators** | `indicators.json` + `pandas-ta` | Configuration-driven chart overlays (SMA/EMA lines) |
| **Database** | PostgreSQL + TimescaleDB | Time-series optimised, mode-namespaced signal/trade tables |
| **Message Broker** | Redis (Pub/Sub) | Decouples ingestion from heavy calculation (future) |
| **Containerisation** | Docker + Docker Compose | Reproducible environment across dev and prod |
| **Reverse Proxy** | Nginx | WebSocket upgrade, load balancing, SSL termination |
| **Configuration** | `backend/config.py` | Single source of truth for all tunables — zero magic numbers |

---

## 2. Project Structure

```
project-oracle/
├── .env                           # Exchange/Brain mode config + API keys
├── .devcontainer/devcontainer.json # VS Code Dev Container specification
├── docker-compose.yml             # PostgreSQL + Redis + PGAdmin
├── indicators.json                # Dynamic Indicator Config (SMA/EMA lines)
├── requirements.txt               # Python dependencies
│
├── backend/                       # Python FastAPI Engine
│   ├── main.py                    # FastAPI app + lifespan manager
│   ├── config.py                  # Centralised HFT tunables (all constants)
│   ├── database.py                # SQLAlchemy async engine + session
│   ├── models.py                  # ORM table definitions (with HFT fields)
│   ├── api/
│   │   ├── analytics.py           # /api/analytics, /api/klines, /api/config/indicators
│   │   ├── ws.py                  # /ws/dashboard signal broadcaster
│   │   └── logger_ws.py           # /ws/logs terminal streamer
│   ├── services/
│   │   ├── ingestion.py           # Dual Binance WS (kline + order book) + scoring loop
│   │   ├── scoring.py             # Legacy scoring engine (preserved for Intraday/Claude)
│   │   └── lifecycle.py           # Active trade monitoring (TP/SL/time decay, fast polling)
│   ├── brain/                     # Multi-brain module
│   │   ├── __init__.py
│   │   ├── market_state.py        # Shared MarketState dataclass (candles, ticks, OB)
│   │   ├── interface.py           # BrainInterface protocol + BrainResult TypedDict
│   │   ├── router.py              # Brain Router — delegates to active brain
│   │   ├── pandas_brain.py        # HFT Scalp Engine (10-component scorer)
│   │   ├── claude_brain.py        # Claude AI implementation (future)
│   │   └── random_brain.py        # Random signal generator (future, DEV ONLY)
│   ├── exchange/                   # Trade execution & safety
│   │   ├── __init__.py
│   │   ├── safety_locks.py        # 4 safety guards (throttle, fee, staleness, time)
│   │   └── trade_executor.py      # REST API order placement (future)
│   ├── static/                    # Legacy static HTML dashboard
│   └── templates/                 # Legacy index.html
│
├── frontend/                      # Angular 19 Workspace
│   ├── src/app/
│   │   ├── chart/chart.ts         # TradingView chart + tier-styled signal lines
│   │   ├── chat/chat.ts           # Signal cards with tier badges + OB gauge
│   │   └── logs/logs.ts           # Dark terminal — live /ws/logs stream
│   ├── src/styles.css             # Global design system
│   └── README.md                  # Frontend-specific documentation
│
└── database/                      # SQL init scripts (future)
    ├── init.sql                   # Full schema creation + hypertables
    └── migrations/
```

---

## 3. The Mode Matrix — Multi-Brain, Multi-Exchange Architecture

### 3.1 Core Concept

The system is governed by two independent toggle switches: **BRAIN MODE** and **EXCHANGE MODE**. Any combination is valid and all produce independently queryable data.

| | Testnet | Live |
|---|---|---|
| **Brain: Pandas Scalp** | Mode A — Scalp + Testnet | Mode B — Scalp + Live |
| **Brain: Claude AI (future)** | Mode C — Claude + Testnet | Mode D — Claude + Live |
| **Brain: Random (future)** | Mode E — Random + Testnet (dev) | **BLOCKED** |

> **Key Design Principle:** Each mode combination writes to its own isolated PostgreSQL table namespace. Random mode is permanently hard-blocked from the Live exchange.

### 3.2 Brain Router — Shared Interface Contract

All brain modules implement the `BrainInterface` protocol. The router, DB writer, and Angular UI never need to know which brain is active — the output shape is always the same.

**`BrainResult` TypedDict** (`backend/brain/interface.py`):

```python
class BrainResult(TypedDict):
    direction: str       # "LONG" | "SHORT"
    score: float         # 0.0 – 100.0 (normalized)
    signal_tier: str     # "STRONG" | "SCALP"
    ob_imbalance: float  # Order book imbalance at signal time (-1 to +1)
    tick_momentum: float # Tick momentum score for analytics
    entry: float         # Entry price
    tp: float            # Take profit
    sl: float            # Stop loss
    time_limit_s: int    # Max trade duration in seconds
    reason: str          # Human-readable signal breakdown
```

### 3.3 Brain Mode Split — Scalp vs Intraday

| | **Scalp Mode (Pandas Brain)** | **Intraday Mode (Claude Brain, future)** |
|---|---|---|
| Trade duration | 1–3 candles (1–3 min) | 15 min to multi-hour |
| Direction source | Micro Trend — last 5 candles | 1H 200 EMA |
| Layer 1 regime | Micro trend + Volatility + Spread | 1H 200 EMA + ADX |
| Scoring engine | 10-component tick-level | Legacy 6-component (scoring.py) |
| TP/SL | Micro-range based (tick buffer) | ATR(14) based |
| Claude API | Not used (too slow) | Primary signal source |

---

## 4. Data Pipeline & WebSocket Architecture

### 4.1 Data Flow

```
EXCHANGE LAYER              BACKEND ENGINE                   FRONTEND (Angular 19)
──────────────              ───────────────────────          ──────────────────────

Binance Kline WS ──────► MarketState                        Angular hooks DIRECTLY
(btcusdt@kline_1m)         ├── candles_buffer (300 deque)   into Binance WS for
  Every tick:              ├── tick_buffer (50 deque)        zero-latency chart
  → tick_buffer            ├── current_price                painting.
  → partial_candle         └── partial_candle
  On x:true only:
  → candles_buffer
  → indicator broadcast

Binance Depth WS ──────► MarketState
(btcusdt@depth5@100ms)     └── orderbook_snapshots (3 deque, anti-spoof)

EMA Cache Loop ────────► MarketState
(every 5 min REST)         └── htf_ema_cached (1H 200 EMA)

Scoring Loop ──────────► Brain Router ──► PandasScalpBrain
(every 2s, throttled)      │                 ├── Layer 1 Gates
                           │                 ├── Layer 2 Scoring (110→100 pts)
                           │                 └── Safety Locks
                           │
                           ├── DB Writer ──► PostgreSQL (AppSignal + UserTrade)
                           │
                           ├── /ws/dashboard ──► app-chart, app-chat
                           │    (signal packets + indicator_update)
                           │
                           └── /ws/logs ──────► app-logs (terminal)
```

### 4.2 Startup Pipeline

1. **Warm-up:** Python fetches the last 300 k-lines via Binance REST API to populate `MarketState.candles_buffer`.
2. **Four concurrent tasks** launch via `asyncio.gather()`:
   - `kline_ws()` — Listens to 1m kline stream. Feeds tick buffer on every message, appends closed candles on `x: true`, broadcasts indicator updates.
   - `orderbook_ws()` — Listens to `depth5@100ms`. Stores last 3 snapshots with float-converted prices/quantities and timestamps.
   - `scoring_loop()` — Polls every 0.5s but only runs scoring if: (a) at least 2s since last score, (b) order book hash changed OR tick buffer has new data. Routes through `brain/router.py`.
   - `htf_ema_cache_loop()` — Fetches 1H 200 EMA every 5 minutes. Writes to `MarketState.htf_ema_cached`. Eliminates REST hammering at tick-level scoring.
3. **Frontend Chart:** Angular hooks directly into Binance WebSocket for zero-latency tick painting, bypassing Python entirely.
4. **Historical Data Proxy:** The `/api/klines` endpoint fetches Binance history, dynamically injects `pandas-ta` indicator values from `indicators.json`.

### 4.3 FastAPI Endpoints

| Endpoint | Purpose |
|---|---|
| `/ws/dashboard` | Broadcasts signal packets (with tier/OB/momentum) + live `indicator_update` payloads |
| `/ws/logs` | Routes `logging.getLogger()` output to the Angular terminal viewer |
| `/api/config/indicators` | Serves `indicators.json` schema to the frontend |
| `/api/klines` | Binance proxy with dynamically computed indicator overlays |
| `/api/analytics` | Dashboard stats (signals, win rate, trade breakdown) |

---

## 5. Dynamic Indicators — Configuration-Driven Architecture

### 5.1 Overview

The system uses `indicators.json` as the single source of truth for chart overlays. Adding a new indicator requires zero code changes — just edit the JSON file and restart.

### 5.2 `indicators.json` Schema

```json
[
  { "id": "ma_20",  "type": "sma", "length": 20,  "color": "#2962FF", "lineWidth": 2 },
  { "id": "ma_50",  "type": "sma", "length": 50,  "color": "#FF9800", "lineWidth": 2 },
  { "id": "ma_200", "type": "sma", "length": 200, "color": "#E0E0E0", "lineWidth": 2 }
]
```

**Supported types:** `sma` (Simple Moving Average), `ema` (Exponential Moving Average).

### 5.3 Pipeline Flow

1. **Backend:** On every candle close, `ingestion.py::_broadcast_indicators()` reads `indicators.json`, computes each indicator via `pandas-ta`, and broadcasts via `/ws/dashboard` as `indicator_update` packets.
2. **REST Proxy:** The `/api/klines` endpoint injects the same indicator values into historical candle payloads.
3. **Frontend:** Angular fetches the schema from `/api/config/indicators`, dynamically creates `chart.addLineSeries()` instances, and renders toggle checkboxes. The UI draws whatever the backend tells it to.

---

## 6. The HFT Scalp Scoring Engine (2-Layer System)

The `PandasScalpBrain` (`backend/brain/pandas_brain.py`) scores continuously at a throttled 2-second interval using tick-level and order book data. A signal is dispatched only if the normalized score meets the tier threshold AND passes all safety guards.

### Layer 1: Gatekeepers (All Must Pass)

| Gate | Method | Data Source | Replaces |
|---|---|---|---|
| **Micro Trend** | HH/HL consecutive-pair check on last 5 candles. Requires 3-of-4 pairs matching. Returns LONG/SHORT/None. | Last 5 closed 1m candles | 1H 200 EMA (for scalp) |
| **Volatility Gate** | `ATR_current / ATR_50_avg` must be in [0.5, 3.0]. Too quiet = can't clear fees. Too chaotic = flash spike. | ATR(14) series | ADX(14) chop filter |
| **Spread Gate** | Best ask - best bid must be < 0.05% of bid. Rejects abnormally wide spreads. | Latest order book snapshot | New (no predecessor) |

### Layer 2: Scoring Engine (110 Raw Points, Normalized to 100)

| Component | Weight | Data Source | Lag | Key Detail |
|---|---|---|---|---|
| **Order Book Imbalance** | 25 pts | Order book depth stream | ~100ms | Averages last 3 snapshots (anti-spoof) |
| **Tick Momentum** | 20 pts | Tick buffer (last 20 ticks) | ~1–5 sec | Bounded `(|late|-|early|)/(|late|+|early|)` formula |
| **Micro Swing S/R** | 15 pts | Last 10 closed 1m candles | ~10 min | Distance-from-swing scoring (15% = full, 30% = half) |
| **Candle Body Pressure** | 10 pts | Current partial candle | Real-time | Body/range ratio aligned with direction |
| **Body Acceleration** | 10 pts | Last 5 candles | ~5 min | Late body avg / early body avg (expanding = energy) |
| **Wick Rejection** | 8 pts | Last 3 candles | ~3 min | Lower wick ratio (LONG) or upper wick ratio (SHORT) |
| **Volume Confirmation** | 7 pts | Last 5 candles | ~5 min | Up-volume vs down-volume ratio |
| **Consecutive Closes** | 5 pts | Last 5 candles | ~5 min | Streak count of closes in direction (0–5) |
| **Round Numbers** | 5 pts | Modulo math | None | Within 0.5% of $1000 round number |
| **Engulfing / Inside Bar** | +5 bonus | Last 2 candles | ~2 min | Bullish/bearish engulfing or inside bar compression |

**Normalization:** `final_score = (raw_total / 110) * 100`

### Signal Tiers

| Tier | Threshold | TP/SL | Time Limit |
|---|---|---|---|
| **STRONG** | Normalized >= 75 | TP = 1.5x micro range, SL = 0.75x (2:1) | 300s (5 min) |
| **SCALP** | Normalized >= 60 | TP = 1.0x micro range, SL = 0.5x (2:1) | 180s (3 min) |
| No Trade | Below 60 | — | — |

### Micro-Range Target Generation

```python
# Uses last 20 ticks from tick_buffer to determine current micro volatility
micro_range = max(recent_prices) - min(recent_prices)
micro_range = max(micro_range, entry * 0.0008)  # Floor: covers 2x Binance fee

# LONG example:
tp = entry + (micro_range * tp_mult)
sl = entry - (micro_range * sl_mult)
```

---

## 7. Safety Guards (`backend/exchange/safety_locks.py`)

Four guards run AFTER scoring produces a result, BEFORE signal emission. All must pass.

| Guard | Logic | Purpose |
|---|---|---|
| **Frequency Throttle** | Max 1 signal per 30 seconds | Prevents overtrading at tick-level scoring |
| **Fee Viability** | `|TP - entry| > entry * 0.001 * 2.5` | Ensures TP clears 2.5x round-trip Binance fees |
| **Order Book Staleness** | Reject if latest OB snapshot > 500ms old | Prevents trading on stale data |
| **Time Limit** | Passed into `BrainResult.time_limit_s` | Enforced by lifecycle monitor |

---

## 8. Centralised Configuration (`backend/config.py`)

All tunables live in one file. No magic numbers scattered across the codebase.

| Category | Key Constants |
|---|---|
| **Data Streams** | `TICK_BUFFER_SIZE=50`, `OB_SNAPSHOT_HISTORY=3`, `OB_STALENESS_MS=500`, `HISTORICAL_CANDLES=300` |
| **Scoring Throttle** | `SCORING_THROTTLE_S=2.0`, `SIGNAL_COOLDOWN_S=30` |
| **Layer 1 Gates** | `MICRO_TREND_CANDLES=5`, `MICRO_TREND_MIN_PAIRS=3`, `VOL_RATIO_MIN=0.5`, `VOL_RATIO_MAX=3.0`, `SPREAD_MAX_PCT=0.05` |
| **Layer 2 Weights** | OB=25, Tick=20, SR=15, Body=10, Accel=10, Wick=8, Vol=7, Consec=5, Round=5, Engulfing=+5 (110 raw) |
| **Signal Tiers** | `STRONG_THRESHOLD=75`, `SCALP_THRESHOLD=60` |
| **Targets** | `STRONG_TP_MULT=1.5`, `STRONG_SL_MULT=0.75`, `SCALP_TP_MULT=1.0`, `SCALP_SL_MULT=0.5`, `MIN_RANGE_PCT=0.0008` |
| **Safety** | `BINANCE_FEE_PCT=0.001`, `FEE_VIABILITY_MULT=2.5`, `SCALP_TIME_LIMIT_S=180`, `STRONG_TIME_LIMIT_S=300` |
| **EMA Cache** | `HTF_EMA_CACHE_TTL_S=300`, `HTF_EMA_LENGTH=200` |
| **Lifecycle** | `LIFECYCLE_POLL_FAST_S=2`, `LIFECYCLE_POLL_SLOW_S=10` |

---

## 9. Database Schema

### 9.1 Table Naming Convention

```sql
-- Pattern: {brain_mode}_{exchange_mode}_{table_type}
-- Examples:
pandas_testnet_signals     pandas_testnet_trades
pandas_live_signals        pandas_live_trades
claude_testnet_signals     claude_testnet_trades
random_testnet_signals     random_testnet_trades   -- DEV ONLY
```

### 9.2 Current Tables

**`app_signals`** — The Brain's output

| Column | Type | Description |
|---|---|---|
| `id` | PK | Unique identifier |
| `pair` | VARCHAR (indexed) | e.g., BTCUSDT |
| `timestamp` | DATETIME | Signal generation time |
| `direction` | VARCHAR | LONG / SHORT |
| `probability_score` | INTEGER | 0–100 (normalized) |
| `suggested_entry_price` | FLOAT | Entry price |
| `suggested_tp` / `suggested_sl` | FLOAT | Take profit / Stop loss |
| `expected_timeframe_mins` | INTEGER | Legacy time limit (minutes) |
| `signal_tier` | VARCHAR (nullable) | STRONG / SCALP |
| `ob_imbalance` | FLOAT (nullable) | Order book imbalance at signal time (-1 to +1) |
| `tick_momentum` | FLOAT (nullable) | Tick momentum score for analytics |
| `trade_duration_s` | INTEGER (nullable) | HFT time limit in seconds |

**`user_trades`** — Execution & Lifecycle

| Column | Type | Description |
|---|---|---|
| `id` | PK | Unique identifier |
| `signal_id` | FK | Links to `app_signals` |
| `status` | VARCHAR | ACTIVE / CLOSED_TP / CLOSED_SL / INVALIDATED |
| `invalidation_reason` | VARCHAR (nullable) | TIME_LIMIT_SCALP / TIME_LIMIT_STRONG / TIME_LIMIT_EXCEEDED |
| `actual_entry/tp/sl` | FLOAT | Trade parameters |
| `close_price` | FLOAT (nullable) | Final close price |
| `realized_pnl` | FLOAT (nullable) | Profit/loss calculation |

---

## 10. Trade Lifecycle & Active Monitoring

1. **Signal:** Scoring loop produces a `BrainResult` that passes all safety guards
2. **Auto-Execute:** `ingestion.py::_emit_signal()` saves `AppSignal` + auto-creates `UserTrade` with status ACTIVE
3. **Broadcast:** Signal payload (with `signal_tier`, `ob_imbalance`, `tick_momentum`) sent via `/ws/dashboard`
4. **Mid-Trade Monitoring:** `lifecycle.py::monitor_active_trades()` runs continuously:
   - **Fast polling (2s)** for SCALP/STRONG tier trades
   - **Slow polling (10s)** for legacy trades
   - Uses `market_state.current_price` (streamed) instead of REST API calls
   - **TP/SL check:** Auto-closes at take profit or stop loss
   - **Time decay:** Seconds-based for HFT (`trade_duration_s`), minutes-based for legacy (`expected_timeframe_mins`)
5. **Trade Resolution:**
   - Auto-trigger: Price hits TP or SL
   - Auto-invalidation: SCALP > 180s or STRONG > 300s without TP hit

---

## 11. Frontend Dashboard — Angular 19

### 11.1 Component Architecture

| Component | Panel | Responsibility |
|---|---|---|
| `app-chart` | Left (60%) | TradingView charts: candles, volume, dynamic indicator overlays, tier-styled signal price lines |
| `app-chat` | Top-Right | Signal cards with tier badges (STRONG gold / SCALP blue), OB imbalance gauge, tick momentum score |
| `app-logs` | Bottom-Right | Dark terminal — live `/ws/logs` stream with severity colour coding |
| `app-mode-control` | Top Bar *(future)* | Brain mode toggle, Exchange mode toggle, active mode badge |

### 11.2 Signal Card Display

Each signal card shows:
- **Pair + Tier Badge** — STRONG (gold gradient) or SCALP (blue gradient)
- **Direction + Score** — LONG/SHORT with normalized score /100
- **Entry / TP / SL** — Monospace formatted prices
- **OB Imbalance Gauge** — Horizontal bar from -1 (sellers) to +1 (buyers), green/red fill
- **Tick Momentum** — Numeric display

### 11.3 Chart Signal Lines

- **STRONG tier:** Solid 2px gold entry line, 2px TP/SL lines
- **SCALP tier:** Dashed 1px blue entry line, 1px TP/SL lines
- TP/SL colours: LONG TP = blue, LONG SL = red (inverted for SHORT)

### 11.4 UI Features

* **`angular-split`** — Drag-to-resize panels (Chart, Chat, Logs)
* **Dynamic Indicator Checkboxes** — Auto-generated from `indicators.json`
* **Time Interval Filters** — 1m, 1h, 4h, 1d buttons
* **Smart Auto-Scroll** — Log viewer scrolls automatically but pauses when user scrolls up

---

## 12. Infrastructure & Containerisation (Docker)

### 12.1 Docker Compose — External Services

`docker-compose.yml` orchestrates:
1. **PostgreSQL (`pyprophet-db-1`)** — Live relational database on internal bridge network
2. **PGAdmin** — Web GUI for database management on port 5050
3. **Redis** *(future)* — Pub/Sub message broker for decoupled ingestion

### 12.2 VS Code Dev Container

`.devcontainer/devcontainer.json` teleports VS Code into an isolated Linux sandbox:
* **OS:** Microsoft `python:1-3.12-bullseye` Debian image
* **Node:** `ghcr.io/devcontainers/features/node:1` for Angular 19 builds
* **Network:** Mapped into `pyprophet_default` bridge network
* **Exposed Ports:** `8000` (FastAPI), `4200` (Angular), `5050` (PGAdmin)
* **Startup:** `uvicorn backend.main:app --host 0.0.0.0 --port 8000`

---

## 13. Phase-by-Phase Implementation Plan

| Phase | Name | Status | Primary Goal |
|---|---|---|---|
| Phase 1 | Foundation | Done | Docker, DB, WebSocket warm-up, indicator pipeline |
| Phase 2 | Legacy Pandas Brain | Done | Original scoring engine (ADX, RSI, VADER mocked) |
| Phase 2.5 | HFT Foundation | Done | Config centralisation, brain/ scaffold, MarketState |
| Phase 3 | HFT Ingestion | Done | Dual WS (kline + order book), tick buffer, scoring loop, EMA cache |
| Phase 3.5 | Scalp Scoring Engine | Done | 10-component scorer, micro-range targets, signal tiers |
| Phase 4 | Safety Layer | Done | Frequency throttle, fee guard, OB staleness, time limits |
| Phase 5 | Frontend HFT | Done | Tier badges, OB gauge, tier-styled chart lines |
| Phase 6 | Claude Brain | Future | Claude integration, prompt design, Intraday mode |
| Phase 7 | Analytics Layer | Future | Multi-mode comparison dashboard, backtesting |
| Phase 8 | Live Graduation | Future | First real scalp trades, conservative sizing |

---

## 14. What Was Preserved (Legacy)

| Component | File | Purpose |
|---|---|---|
| Original scoring engine | `backend/services/scoring.py` | ADX + 1H EMA + RSI + mocked VADER/patterns — kept for future Intraday/Claude mode |
| Dynamic indicator pipeline | `indicators.json` | Unchanged — still drives chart overlays |
| Docker Compose setup | `docker-compose.yml` | No changes |
| Dev Container config | `.devcontainer/` | No changes |
| Angular panel layout | `angular-split` | No changes |
| TradingView integration | `chart.ts` base | Only styling additions |
| `/ws/logs` terminal | `logger_ws.py` + `logs.ts` | No changes |

---

## 15. Risk Management

### Non-Negotiable Safety Rules

1. Random brain mode is permanently blocked from Live exchange (future: 3 independent guards)
2. 1 active trade per pair at any time — database unique constraint
3. Binance API key must **NEVER** have withdrawal permissions
4. Binance API key must be IP-whitelisted
5. Anthropic API monthly spend cap set before live trading
6. All API keys in `.env` only — never hardcoded, never in git
7. Hard stop-loss enforced regardless of brain output
8. **Frequency throttle:** Max 1 signal per 30 seconds (prevents overtrading)
9. **Fee viability guard:** TP must clear 2.5x round-trip Binance fees
10. **Order book staleness guard:** Reject signals if OB data > 500ms old
11. **Time limits:** SCALP = 180s, STRONG = 300s auto-close

### When to Pause Live Trading

* 3 consecutive losing trades in any live mode -> auto-pause
* Claude API returns non-JSON 3 times -> fallback to HOLD
* WebSocket gap-fill detects >5 missed candles
* BTC price move >5% in under 10 minutes (flash crash detection)
* ATR volatility ratio > 3.0 (volatility gate blocks scoring automatically)
