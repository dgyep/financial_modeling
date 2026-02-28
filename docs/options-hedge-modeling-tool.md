# Options Hedge Modeling Tool

## Overview

A financial modeling tool for planning and analyzing options-based hedging strategies to protect a large stock position without selling the underlying shares. The core thesis is that the stock trends upward over time but exhibits high volatility, creating downside risk from large swings. The tool models various options strategies—primarily protective puts, covered calls, and collars—to evaluate cost, protection levels, and trade-offs across different market scenarios.

### Target Position

| Detail | Value |
|---|---|
| **Ticker** | MPWR (Monolithic Power Systems) |
| **Shares held** | 925 |
| **Options contracts** | 9 (each contract = 100 shares; 9 contracts cover 900 of 925 shares) |
| **Uncovered shares** | 25 (remaining shares not covered by full contracts) |
| **Dividend** | ~$1.50/share per quarter (~$6.00/year), trending upward |
| **Quarterly dividend income** | ~$1,387.50 (925 × $1.50) |

> **Note**: With 925 shares and 9 contracts, 25 shares remain unhedged by options. The tool should clearly surface this gap in all strategy outputs.

---

## Problem Statement

- You hold 925 shares of **MPWR** and want to **protect against significant downside moves** without liquidating the position.
- Selling the stock is undesirable (long-term bullish conviction; roughly half the shares carry large capital gains, half have minimal gains at current prices).
- MPWR is expected to trend upward over the next couple of years but is highly volatile with potential for large downside swings.
- The strategy goal: **protect the downside, capture most of the upside, and accept giving up gains if the stock swings up rapidly**.
- Options positions will generally be **closed before expiration** to retain the underlying stock (not exercised/assigned). This means modeling should account for closing option positions at market value near expiration rather than assuming exercise.
- The stock pays a quarterly dividend (~$1.50/share, trending higher), which introduces **early assignment risk** on covered calls near ex-dividend dates.
- You need a tool to **model and compare** various options-based protection strategies before executing trades.

---

## Core Strategies to Model

### 1. Protective Put (Long Put)

- **Concept**: Buy put options on the underlying stock to establish a price floor.
- **Parameters to model**:
  - Strike price selection (ATM, OTM at various levels)
  - Expiration date / duration of protection
  - Cost of the put (premium paid)
  - Effective floor price (strike minus premium)
  - Break-even analysis
  - Rolling strategy (cost of continuous protection over time)

### 2. Covered Call (Short Call)

- **Concept**: Sell call options against the existing position to generate premium income.
- **Parameters to model**:
  - Strike price selection (OTM at various levels)
  - Expiration date
  - Premium received
  - Cap on upside (theoretical max if called away; in practice, calls will be closed before expiration)
  - Probability of assignment at various strike levels
  - **Early assignment risk near ex-dividend dates** (MPWR pays ~$1.50/quarter; deep ITM calls are at risk)
  - **Close-before-expiration modeling**: cost to buy back the short call at various stock prices near expiration
  - Rolling strategy and income projection over time

### 3. Collar (Protective Put + Covered Call)

- **Concept**: Simultaneously buy a protective put and sell a covered call. The call premium partially or fully offsets the put cost, creating a defined range of outcomes.
- **Parameters to model**:
  - Put strike (floor) and call strike (ceiling)
  - Net cost (or net credit) of the collar
  - Zero-cost collar identification (where call premium = put premium)
  - Range of outcomes: max loss, max gain, break-even
  - Asymmetric collars (wider upside vs. tighter protection, or vice versa)
  - Rolling and adjusting over time
  - **Dividend impact**: Net cost/credit should account for dividend income retained vs. assignment risk
  - **Close-before-expiration**: Model P&L assuming both legs are closed rather than exercised/assigned

### 4. Put Spread Collar (Future Enhancement)

- **Concept**: Replace the single long put with a put spread to reduce cost, accepting a limited protection range.
- Trade-off: cheaper protection but with a defined maximum protection amount.

### 5. Rolling Over Positions as the Stock Moves

Rolling means closing an existing options position and simultaneously opening a new one — typically at different strikes, a later expiration, or both. This is central to actively managing a hedge over time rather than setting it and forgetting it.

#### Rolling Up (Stock Price Increases)

When MPWR appreciates, the existing hedge becomes misaligned — the put floor is now far below the current price, and the short call may be approaching or in the money.

- **Roll the collar up**: Close both legs and re-establish at higher strikes centered around the new price.
  - **Why**: Locks in a portion of the gain by raising the put floor. Re-centers the protection range.
  - **Cost**: The existing put has lost value (now deeper OTM) and the short call has gained value (now closer to ITM or ITM), so closing the collar has a net cost. The new collar at higher strikes may partially offset this, but rolling up after a significant move **typically costs money**.
  - **Trade-off**: You lock in gains but pay for the privilege. The alternative is to let the existing collar ride and accept the wider gap between current price and the put floor.
- **Roll only the call up**: If the stock is approaching the call strike, buy back the short call and sell a new one at a higher strike.
  - **Why**: Preserves more upside participation without getting called away (or needing to close at a loss on the call leg).
  - **Cost**: Net debit (buying back a more expensive call, selling a cheaper higher-strike call). May be partially offset by time value in the new call.
- **When to roll up**: The tool should model trigger points — e.g., "roll when stock is within X% of the call strike" or "roll when unrealized gain on the position exceeds Y%."

#### Rolling Down (Stock Price Decreases)

When MPWR declines, the dynamics are different and rolling down is more nuanced.

- **The protective put is working**: If the stock drops, the long put gains value — it's doing its job. There's less urgency to act.
- **Roll the put down (generally not advisable)**:
  - Closing a now-valuable ITM put and buying a cheaper lower-strike put would **realize a gain on the put** but **lower the protection floor**, accepting more downside risk from the new level.
  - This is essentially taking profits on the hedge and weakening protection — counter to the strategy's goal.
  - **When it might make sense**: If you believe the decline is temporary and want to harvest put gains while maintaining *some* protection at the lower level.
- **Roll the call down**:
  - The short call has lost value (stock moved away from it). You can buy it back cheaply and sell a new call at a lower strike to collect more premium.
  - **Why**: Generates additional income to offset the cost of the put or to fund a roll of the put to a later expiration.
  - **Risk**: Lowers the upside cap. If the stock rebounds sharply, you're capped at a lower price.
- **Roll both legs down (re-center the collar lower)**:
  - Close both legs — realize a gain on the put, small cost on the call — and re-establish a collar around the new lower price.
  - **When it makes sense**: If the decline is believed to be a new baseline (not a temporary dip) and you want protection centered around the new level.
  - **When it doesn't**: If you expect a rebound, keeping the existing collar is better — the put protects the downside, and the higher call strike gives room for recovery.

#### Summary: Rolling Direction Trade-offs

| Direction | Action | Benefit | Cost / Risk |
|---|---|---|---|
| **Up — full collar** | Close both legs, re-open at higher strikes | Locks in gains, raises floor | Net debit to roll; pays for protection at higher level |
| **Up — call only** | Buy back call, sell higher-strike call | More upside room, avoids assignment | Net debit on the call roll |
| **Down — put only** | Sell valuable put, buy lower-strike put | Harvests put profit | Lowers protection floor; weakens hedge |
| **Down — call only** | Buy back call, sell lower-strike call | Generates income | Lowers upside cap; risk if stock rebounds |
| **Down — full collar** | Close both legs, re-open at lower strikes | Re-centers protection around new price | Accepts the loss as a new baseline; lower ceiling |

#### Key Modeling Requirements for Rolling

- **Cost-to-roll calculator**: Given current positions and a target roll (up/down, one or both legs), calculate the net debit or credit.
- **Trigger-based scenarios**: Model "roll when stock hits $X" or "roll when delta on the call exceeds Y."
- **Cumulative roll cost tracking**: Over multiple quarters, what is the total cost of rolling? How does this compare to the protection gained?
- **Comparison: roll vs. hold**: At each decision point, show the P&L of rolling vs. letting the existing collar ride to expiration (then closing).

---

## Scenario Modeling

The tool should allow the user to define and compare scenarios across these dimensions:

### Stock Price Scenarios
- **Base case**: Stock appreciates at an expected annual rate
- **Bull case**: Stock appreciates faster than expected
- **Bear case**: Stock declines by a defined percentage (e.g., -20%, -30%, -50%)
- **High-volatility sideways**: Stock swings ±X% but ends near current price
- **Custom scenarios**: User-defined price paths or terminal values

### Time Horizons
- Short-term (1–3 months)
- Medium-term (3–12 months)
- Long-term (1–2+ years, modeled as a series of rolled positions)

### Key Outputs per Scenario
- **P&L of the hedged position** vs. unhedged
- **Cost of protection** (absolute and as % of position value)
- **Max downside exposure** with hedge in place
- **Upside participation** (how much upside is capped or retained)
- **Net premium spent or received** over the modeled period
- **Greeks snapshot** (delta, gamma, theta, vega of the combined position)

---

## Options Data Sources

End-of-day or delayed data is acceptable for modeling purposes. Real-time data is not required at this stage.

### E-Trade API (Confirmed Available)

- **Access**: E-Trade customers can request API keys directly from their account at [developer.etrade.com](https://developer.etrade.com).
- **Key type**: An *Individual Key* is tied to your E-Trade user ID and works across all of your E-Trade accounts. No vendor approval needed.
- **Setup steps**:
  1. Complete the [User Intent Survey](https://us.etrade.com/etx/ris/apisurvey/)
  2. Sign the API Agreement
  3. Sign the Market Data Agreement
  4. Request a Sandbox key for development, then a Live key for production
- **Capabilities**: REST API providing account data, quotes, **option chains** (calls, puts, by expiration), and order placement.
- **Authentication**: OAuth 1.0a per-session authorization.
- **Considerations**: Requires annual Market Data Attestation renewal. Well-documented but OAuth flow adds integration complexity.

### Tradier API

- **Access**: Free sandbox account available; brokerage account required for real-time data; delayed data (15 min) may be available without a funded account.
- **Options data**: Full option chain support with **Greeks and implied volatility** included (sourced from ORATS).
- **Pricing**: Free sandbox tier for development. Live market data with a brokerage account (can be unfunded for delayed data — needs verification).
- **Advantages**: Clean REST API, Greeks included natively, good developer documentation.
- **Website**: [documentation.tradier.com](https://documentation.tradier.com/)

### Polygon.io

- **Free tier**: Basic stock data with limited API calls. **Options data requires a paid plan.**
- **Stocks Starter ($29/mo)**: Unlimited API calls, 5 years of historical data, 15-minute delayed quotes. Focused on stock data.
- **Options plans**: Options-specific data starts at higher tiers (~$199/mo for Options Advanced).
- **Considerations**: Excellent data quality and API design, but options data is expensive for a modeling/planning tool.
- **Website**: [polygon.io](https://polygon.io/)

### Other Data Sources to Consider

| Provider | Options Data | Pricing | Notes |
|---|---|---|---|
| **CBOE DataShop** | EOD options summaries | Pay-per-download | Official exchange data, good for historical analysis |
| **EODHD (EOD Historical Data)** | Options chains, historical | Free tier (20 calls/day); paid from ~$20/mo | Decent coverage, affordable |
| **Alpha Vantage** | Limited options support | Free tier available | Better for equities than options |
| **Yahoo Finance (yfinance)** | Options chains | Free (unofficial scraping) | No SLA, may break; good for prototyping |
| **ORATS** | Deep options analytics | Paid | Powers Tradier's Greeks; advanced analytics |

### Recommended Starting Approach

1. **Prototype with Yahoo Finance (yfinance)** — free, easy to integrate via Python, sufficient for initial modeling.
2. **Evaluate E-Trade API** — you already have an account; request an Individual API key to see what data is available at no extra cost.
3. **Consider Tradier** for a cleaner developer experience with built-in Greeks if E-Trade proves cumbersome.

---

## Technical Architecture (Preliminary)

### Platform
- **Language**: Python (rich ecosystem for financial modeling: pandas, numpy, scipy)
- **UI**: TBD — candidates include:
  - Jupyter notebooks for rapid prototyping
  - Streamlit or Gradio for lightweight interactive web UI
  - React-based dashboard for a more polished tool (later phase)

### Core Modules

```
financial_modeling/
├── docs/                        # Documentation (this file)
├── src/
│   ├── data/                    # Market data fetching and caching
│   │   ├── options_chain.py     # Fetch option chains from configured provider
│   │   ├── stock_quotes.py      # Fetch stock price data
│   │   └── cache.py             # Local caching to avoid redundant API calls
│   ├── models/                  # Strategy modeling
│   │   ├── protective_put.py
│   │   ├── covered_call.py
│   │   ├── collar.py
│   │   └── scenario.py          # Scenario definition and evaluation
│   ├── analytics/               # Greeks, P&L, visualization
│   │   ├── greeks.py
│   │   ├── pnl.py
│   │   └── charts.py
│   └── config.py                # API keys, provider selection, defaults
├── tests/
├── requirements.txt
└── README.md
```

### Data Flow

1. **Input**: User specifies stock ticker, position size, and strategy parameters.
2. **Fetch**: Pull current/recent option chain data from the configured provider.
3. **Model**: Calculate P&L across user-defined scenarios for each strategy.
4. **Compare**: Side-by-side output of strategies showing cost, protection range, and upside cap.
5. **Visualize**: Charts showing payoff diagrams, P&L curves, and cost-of-protection over time.

---

## Resolved Questions

1. **Position details**: 925 shares of MPWR, 9 contracts. Single stock focus (MPWR only for now).
2. **Capital gains**: Roughly half the shares have large capital gains, half have minimal gains at current price. This reinforces the desire not to sell.
3. **Dividend impact**: MPWR pays ~$1.50/share/quarter, trending upward. Early assignment risk on covered calls near ex-dividend dates must be modeled.
4. **Exercise vs. close**: Options will generally be **closed before expiration** to retain stock positions, not held to exercise/assignment. The tool should model closing costs.
5. **Rolling strategy**: Single expiration modeling first. Quarterly rollovers are the likely real-world cadence; rolling cost modeling is a natural Phase 4 extension.
6. **Historical backtesting**: Desired, but should not delay development of forward-looking scenario modeling. Developed in parallel or as a fast-follow.
7. **Investment thesis**: Stock expected to rise over 2+ year horizon. High volatility with wide downside swings. Strategy: protect downside, capture most upside, accept giving up rapid upside gains.

## Remaining Open Questions

1. **Tax considerations**: Should the tool factor in tax implications of options activity (e.g., wash sale rules, short-term vs. long-term treatment of premiums)? *(Noted: ~half the position has significant capital gains.)*
2. **Alerts / monitoring**: Eventual feature — scope and priority TBD.

---

## Phased Roadmap (Proposed)

| Phase | Scope | Deliverable |
|---|---|---|
| **Phase 1** | Core modeling engine | Protective put, covered call, and collar P&L modeling for 9 contracts of MPWR with manual data input or yfinance. Single expiration. Account for 25 uncovered shares and dividend income. |
| **Phase 2** | Data integration | Connect to E-Trade or Tradier API for live option chain ingestion. Model ex-dividend early assignment risk. |
| **Phase 3** | Scenario comparison | Side-by-side strategy comparison with visualization (payoff diagrams, cost charts) |
| **Phase 3b** | Historical backtesting | Replay strategies against MPWR historical price/options data. Developed in parallel with Phase 3, not blocking it. |
| **Phase 4** | Rolling & time-series | Model ongoing cost of quarterly collar/put rolls over multi-quarter horizons |
| **Phase 5** | UI & usability | Interactive dashboard (Streamlit or web-based) for non-technical use |
