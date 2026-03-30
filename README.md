# PyProphet

An institutional-grade, real-time algorithmic trading and backtesting engine engineered for high-throughput cryptocurrency markets (specifically Binance). 

Unlike traditional "fire-and-forget" retail trading bots that blindly execute logic based on lagging crossover indicators, PyProphet is built around a dynamic, probabilistic **Two-Layer Scoring Engine** and **Active Trade Lifecycle Monitoring**.

The system treats every market movement as a fluid probability rather than a binary trigger, continuously re-evaluating the validity of a trade even after execution.

---

## Technical Architecture

PyProphet is engineered with a modern, decoupled tech stack to ensure zero-latency execution and strict data separation:

*   **Backend Engine:** Built on asynchronous **Python (FastAPI)**. Maintains a persistent WebSocket connection to Binance to react instantly to closed 1-minute payloads.
*   **Data Processing:** Leverages **Pandas** and **Pandas-TA** to maintain in-memory circular arrays of price data, efficiently computing technicals without recalculating historical datasets on every tick.
*   **Database:** A **PostgreSQL** relational database structured to strictly partition the system's *theoretical signals* from the user's *actual execution* to perform granular forensics on slippage and strategy decay.
*   **Client Interface:** A scalable **Angular 19** frontend featuring TradingView Lightweight Charts and RxJS to stream live WebSocket probability updates and structural backend server logs seamlessly to the client.

---

## The Two-Layer Scoring Engine

PyProphet computes an aggregate probability score (0-100%). A trade signal is strictly dispatched only if the cumulative score meets or exceeds a **70% threshold**.

1. **Layer 1 (The Gatekeepers):** Strict baseline filters. A failure here instantly kills the setup.
   * *Volatility Filter:* Neutralizes the engine if ADX < 20 (chopping regime).
   * *Macro Trend Alignment:* Ensures 1m signals never fight the prevailing 1H 200 EMA structure.
2. **Layer 2 (The Weighted Probability Matrix):** If Gatekeepers are passed, the algorithm scores the setup across a diverse matrix:
   * *Macro Foundation (25%):* 200 EMA alignment.
   * *Machine Learning Nodes (20%):* Using K-Means Clustering to dynamically identify heavy consolidation support/resistance nodes mathematically.
   * *Momentum Extremes (20%):* RSI utilized purely as ultimate triggers during extreme overbought/oversold exhaustion.
   * *Semantic Sentiment Analysis (15%):* VADER NLP scoring against crypto news aggregators.
   * *Pattern Recognition (10%):* Percentage-based variance matching for chart structures (Flags, Engulfing).
   * *Psychological Walls (10%):* Modulo math arrays to detect proximity buffers around clean, psychological round numbers (e.g., $70,000).

---

## Active Mid-Trade Monitoring

Standard retail bots open a trade with a Take Profit (TP) and Stop Loss (SL) and go dormant. PyProphet actively monitors the *health* of a trade while it is open. 

Because the backend WebSocket continuously re-scores the live asset, the system can proactively detect if the foundational thesis of the trade has broken. If probability drops below 50% due to:
*   A sudden drop in Volume / VWAP
*   The breakdown of the structural flag/pattern
*   A violent counter-trend price spike
*   "Time-to-target" exhaustion limits

PyProphet will auto-invalidate and prematurely close the trade, protecting capital before the hard Stop Loss is hit, logging the exact invalidation reason into PostgreSQL for subsequent forensic backtesting optimization.

---

## Business Strategy & Monetization Models

PyProphet is uniquely positioned to bypass the massive churn of standard retail B2C trading apps. 

### Strategy A: The "Signal-as-a-Service" Freemium Model
**Do not charge users to execute trades.** Charge them for the *intel*.
*   **Free Tier:** Users get the live Angular Web App with delayed signals. They can paper trade. This hooks them on the UI.
*   **Pro Tier ($29/month):** Real-time WebSocket signals for the 1-minute timeframe with live UI pop-ups.
*   **Institutional Tier ($199/month):** Unlocks the API so enterprise clients can plug their own automated trading bots via Webhook directly into PyProphet's JSON signal stream.

### Strategy B: The "Copy-Trading" Profit Share
Instead of a flat monthly fee, take a percentage of the profits generated, perfectly aligning incentives.
*   Users connect their Binance API keys for free. 
*   At the end of every month, if the user made $1,000 in net profit because of the bot, the software automatically invoices them for a **15% performance fee** ($150). If they lose money that month, they pay nothing.

### Strategy C: The B2B Whitelabel
Avoid building a massive B2C app entirely.
*   Build a sleek API dashboard and sell the "Oracle Scoring Engine" to *other* crypto platforms (Prop Firms, DEXs, Discord trading groups). 
*   Charge Discord server owners $500/month to integrate the Python backend via Webhook to broadcast ">85% Probability (K-Means Support hit)" signals directly to their VIP members, managing zero retail users in the process.
