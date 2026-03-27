# Trading Strategies

This document catalogs our trading strategies, explains how we discover and collect them, and defines the evaluation framework used to decide which strategies move from idea to production.

---

## Table of Contents

1. [Strategy Categories](#strategy-categories)
2. [Strategy Catalog](#strategy-catalog)
3. [How to Collect Strategies](#how-to-collect-strategies)
4. [Evaluation Framework](#evaluation-framework)
5. [Strategy Lifecycle](#strategy-lifecycle)
6. [Risk Classification](#risk-classification)

---

## Strategy Categories

### 1. Technical Analysis (TA)

Strategies based on price action, volume, and mathematical indicators.

| Strategy | Description | Markets | Complexity |
|----------|-------------|---------|-----------|
| **Moving Average Crossover** | Buy when fast MA crosses above slow MA, sell on reverse | Crypto | Low |
| **RSI Mean Reversion** | Buy when RSI < 30 (oversold), sell when RSI > 70 (overbought) | Crypto | Low |
| **Bollinger Band Squeeze** | Enter on breakout after low-volatility squeeze | Crypto | Medium |
| **MACD Divergence** | Detect divergence between price and MACD for reversal signals | Crypto | Medium |
| **VWAP Reversion** | Trade reversion to Volume Weighted Average Price | Crypto | Medium |
| **Ichimoku Cloud** | Multi-signal system using cloud, conversion, base lines | Crypto | High |

### 2. AI/ML-Based

Strategies that use machine learning models for signal generation.

| Strategy | Description | Markets | Complexity |
|----------|-------------|---------|-----------|
| **Sentiment Analysis** | Analyze news/social media sentiment to predict price movement | Crypto, Polymarket | High |
| **Price Prediction (LSTM)** | Time-series forecasting using Long Short-Term Memory networks | Crypto | High |
| **Classification Signals** | Random Forest / XGBoost classifying next-candle direction | Crypto | Medium |
| **Reinforcement Learning** | Agent learns optimal trading policy through reward maximization | Crypto | Very High |
| **LLM Event Analysis** | Use LLMs to parse news events and predict Polymarket outcomes | Polymarket | High |

### 3. Market-Making & Arbitrage

Strategies focused on providing liquidity or exploiting price differences.

| Strategy | Description | Markets | Complexity |
|----------|-------------|---------|-----------|
| **Cross-Exchange Arbitrage** | Exploit price differences between exchanges | Crypto | Medium |
| **Triangular Arbitrage** | Profit from price inconsistencies across 3 trading pairs | Crypto | High |
| **Grid Trading** | Place buy/sell orders at regular intervals around a price | Crypto | Medium |
| **Polymarket Arbitrage** | Exploit mispricing between correlated Polymarket events | Polymarket | High |

### 4. Prediction Market Specific

Strategies tailored for binary/multi-outcome prediction markets.

| Strategy | Description | Markets | Complexity |
|----------|-------------|---------|-----------|
| **Probability Mispricing** | Identify events where market odds diverge from model probability | Polymarket | High |
| **News-Driven Trading** | React to breaking news faster than the market adjusts | Polymarket | High |
| **Portfolio of Bets** | Diversified portfolio of uncorrelated prediction market positions | Polymarket | Medium |
| **Contrarian Sentiment** | Bet against extreme public sentiment when data suggests otherwise | Polymarket | High |

---

## How to Collect Strategies

### Sources

| Source | What to Extract | How |
|--------|----------------|-----|
| **Academic Papers** | Peer-reviewed strategies with backtested results | arxiv.org (q-fin), SSRN, Journal of Finance |
| **Trading Books** | Classic and modern strategy patterns | "Advances in Financial ML" (De Prado), "Algorithmic Trading" (Chan) |
| **Open-Source Projects** | Tested implementations and ideas | GitHub (freqtrade, jesse, hummingbot) |
| **Trading Communities** | Emerging ideas, real-world feedback | Reddit (r/algotrading), QuantConnect, Elite Trader |
| **Crypto-Specific Research** | DeFi strategies, MEV, on-chain signals | Dune Analytics, DefiLlama, Messari |
| **Polymarket Discord/Docs** | Market mechanics, API capabilities, community insights | Polymarket docs, Discord |
| **LLM-Assisted Research** | Rapid literature review, strategy brainstorming | Claude, GPT-4 for summarizing papers |

### Collection Process

```
1. DISCOVER    →  Find strategy idea from source above
2. DOCUMENT    →  Write it up in strategy template (see below)
3. QUICK TEST  →  Napkin math: does the edge make sense?
4. BACKTEST    →  Run on historical data
5. REVIEW      →  Evaluate with framework (see below)
6. DECIDE      →  Promote, iterate, or discard
```

### Strategy Template

Every new strategy must be documented using this template before any code is written:

```markdown
# Strategy: [Name]

## Summary
One paragraph describing the strategy.

## Hypothesis
What market inefficiency does this exploit? Why should this edge exist?

## Rules
- Entry signal: ...
- Exit signal: ...
- Position sizing: ...
- Stop-loss: ...
- Take-profit: ...

## Markets
- [ ] Crypto (which pairs?)
- [ ] Polymarket (which event types?)

## Data Requirements
- What data feeds are needed?
- What timeframes?
- Any alternative data (sentiment, on-chain)?

## Expected Performance
- Target win rate: ...
- Target risk/reward ratio: ...
- Target Sharpe ratio: ...
- Maximum drawdown tolerance: ...

## Risks & Limitations
- What can go wrong?
- Market conditions where this fails?
- Overfitting concerns?

## References
- Papers, articles, or code that inspired this strategy
```

---

## Evaluation Framework

Every strategy is scored on these dimensions before promotion:

### Quantitative Metrics (from backtest)

| Metric | Description | Minimum Threshold |
|--------|-------------|-------------------|
| **Sharpe Ratio** | Risk-adjusted return (annualized) | > 1.5 |
| **Sortino Ratio** | Downside-risk-adjusted return | > 2.0 |
| **Max Drawdown** | Largest peak-to-trough decline | < 20% |
| **Win Rate** | Percentage of profitable trades | > 45% |
| **Profit Factor** | Gross profit / Gross loss | > 1.5 |
| **Total Return** | Cumulative PnL over backtest period | Positive |
| **Number of Trades** | Statistical significance | > 100 |
| **Calmar Ratio** | Annual return / Max drawdown | > 1.0 |

### Qualitative Assessment

| Factor | Questions to Answer |
|--------|-------------------|
| **Edge Validity** | Is the edge logical? Will it persist? |
| **Market Regime** | Does it work in trending AND ranging markets? |
| **Execution Feasibility** | Can we execute at the speed/cost required? |
| **Data Availability** | Can we reliably source the required data? |
| **Scalability** | Does the edge disappear with larger positions? |
| **Correlation** | Is it uncorrelated with our existing strategies? |

### Scoring Matrix

Each strategy gets a score from 1-5 on each dimension:

```
Total Score = (Sharpe × 2) + (Drawdown × 2) + (Edge Validity × 3)
            + (Execution × 1) + (Scalability × 1) + (Correlation × 1)

Max Score: 50
Promotion Threshold: 35+
```

---

## Strategy Lifecycle

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│   IDEA   │───▶│ RESEARCH │───▶│ BACKTEST │───▶│  PAPER   │───▶│   LIVE   │
│          │    │          │    │          │    │ TRADING  │    │ TRADING  │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
     │               │               │               │               │
     ▼               ▼               ▼               ▼               ▼
  Template      Literature       Historical      Simulated       Real capital
  filled out    review done      data tested     with live       deployed
                                                 market data

  Gate: Has     Gate: Edge      Gate: Meets     Gate: Matches    Gate: Risk
  a clear       makes sense     quantitative    backtest         limits
  hypothesis    theoretically   thresholds      expectations     respected
```

### Stage Definitions

| Stage | Duration | Capital at Risk | Exit Criteria |
|-------|----------|-----------------|---------------|
| **Idea** | 1-2 days | $0 | Hypothesis documented |
| **Research** | 3-5 days | $0 | Literature review complete, edge validated |
| **Backtest** | 1-2 weeks | $0 | Passes quantitative thresholds |
| **Paper Trading** | 2-4 weeks | $0 (simulated) | Live results match backtest within 20% |
| **Live Trading** | Ongoing | Real capital (start small) | Continuous monitoring, kill switch active |

---

## Risk Classification

Each strategy is assigned a risk tier that determines capital allocation limits:

| Tier | Max Capital Allocation | Max Leverage | Description |
|------|----------------------|-------------|-------------|
| **Conservative** | 30% of portfolio | 1x | Proven strategies with long track records |
| **Moderate** | 15% of portfolio | 2x | Backtested strategies with 4+ weeks paper trading |
| **Aggressive** | 5% of portfolio | 3x | New strategies with limited live track record |
| **Experimental** | 1% of portfolio | 1x | Strategies still being validated |

### Risk Rules (Non-Negotiable)

1. **Never risk more than 2% of total portfolio on a single trade**
2. **Global stop-loss**: If portfolio drops 10% in a day, halt ALL strategies
3. **Strategy stop-loss**: If a strategy hits 15% drawdown, pause and review
4. **Correlation limit**: No two active strategies with correlation > 0.7
5. **Kill switch**: Every strategy must have an instant shutdown mechanism
