# POC → Development → Production Pipeline

This document defines how a trading strategy (or any feature) moves from an initial idea through to running in production with real capital. Every phase has clear entry/exit criteria and deliverables.

---

## Table of Contents

1. [Pipeline Overview](#pipeline-overview)
2. [Phase 1: POC (Proof of Concept)](#phase-1-poc-proof-of-concept)
3. [Phase 2: Development](#phase-2-development)
4. [Phase 3: Staging / Paper Trading](#phase-3-staging--paper-trading)
5. [Phase 4: Production](#phase-4-production)
6. [Automated Promotion Pipeline](#automated-promotion-pipeline)
7. [Rollback & Kill Switch](#rollback--kill-switch)
8. [Checklists](#checklists)

---

## Pipeline Overview

```
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│   POC   │────▶│   DEV   │────▶│ STAGING │────▶│  PROD   │
│         │     │         │     │ (Paper) │     │ (Live)  │
└─────────┘     └─────────┘     └─────────┘     └─────────┘
  Jupyter         Git branch      Paper trading    Real capital
  Notebook        CI/CD           Simulated exec   Monitored
  Quick test      Full tests      Performance      Alerting
                  Code review     validation       Kill switch

  Duration:       Duration:       Duration:        Duration:
  1-3 days        1-2 weeks       2-4 weeks        Ongoing

  Risk: $0        Risk: $0        Risk: $0         Risk: Real
```

---

## Phase 1: POC (Proof of Concept)

### Goal
Quickly validate whether a strategy idea has potential **before investing time in proper engineering**.

### Environment
- **Where**: Local machine, Jupyter notebook
- **Data**: Downloaded historical data (CSV or API)
- **Execution**: Simulated (no exchange connection)
- **Code quality**: Doesn't matter — this is throwaway code

### What to Do

```
1. Write strategy logic in a Jupyter notebook
2. Download historical data for target market
3. Run a simple backtest (vectorized or event-driven)
4. Plot equity curve, compute basic metrics
5. Decide: promote, iterate, or kill
```

### Entry Criteria
- [ ] Strategy template filled out (see [Strategy Docs](../strategies/README.md))
- [ ] Clear hypothesis about what market inefficiency this exploits
- [ ] Historical data available for target market

### Exit Criteria (to promote to DEV)
- [ ] Sharpe ratio > 1.0 on historical data
- [ ] Profit factor > 1.2
- [ ] Max drawdown < 30%
- [ ] At least 50 trades in backtest
- [ ] The edge makes logical sense (not just curve-fitting)

### Deliverables
- Jupyter notebook with backtest results
- One-page summary: hypothesis, results, decision
- If promoted: GitHub issue created for DEV phase

### Anti-Patterns to Avoid
- **Overfitting**: Don't optimize parameters until the backtest looks perfect
- **Survivorship bias**: Make sure your data includes delisted tokens/resolved markets
- **Look-ahead bias**: Don't use future data in your signals
- **Too few trades**: 10 winning trades doesn't prove anything

---

## Phase 2: Development

### Goal
Turn the validated POC into **production-quality code** with proper tests, documentation, and integration.

### Environment
- **Where**: Git feature branch, local Docker Compose
- **Data**: TimescaleDB (local) with historical data
- **Execution**: Simulated execution engine
- **Code quality**: Full test coverage, type hints, linting

### What to Do

```
1. Create feature branch: feature/strategy-{name}
2. Implement strategy as a plugin (following strategy interface)
3. Write unit tests for strategy logic
4. Write integration tests with simulated market data
5. Run full backtest suite (not just quick POC test)
6. Run against multiple time periods and market conditions
7. Code review (PR)
8. Merge to develop
```

### Code Structure for a New Strategy

```typescript
// apps/strategy-engine/src/strategies/rsi-mean-reversion.ts

import { BaseStrategy, Signal, type Candle, type Portfolio } from '@polytrade/shared-types';

export class RSIMeanReversion extends BaseStrategy {
  readonly name = 'rsi_mean_reversion';
  readonly version = '1.0.0';
  readonly markets = ['crypto'] as const;

  private rsiPeriod: number;
  private oversold: number;
  private overbought: number;

  constructor(config: { rsiPeriod?: number; oversold?: number; overbought?: number }) {
    super();
    this.rsiPeriod = config.rsiPeriod ?? 14;
    this.oversold = config.oversold ?? 30;
    this.overbought = config.overbought ?? 70;
  }

  onCandle(candle: Candle): Signal | null {
    const rsi = this.indicators.rsi(candle.close, this.rsiPeriod);

    if (rsi < this.oversold) return Signal.BUY;
    if (rsi > this.overbought) return Signal.SELL;
    return null;
  }

  getPositionSize(_signal: Signal, portfolio: Portfolio): number {
    return portfolio.equity * 0.02; // Risk 2% per trade
  }
}
```

### Testing Requirements

```
apps/strategy-engine/
├── src/strategies/
│   └── rsi-mean-reversion.ts
├── tests/
│   ├── unit/
│   │   └── rsi-mean-reversion.test.ts     # Pure logic tests (vitest)
│   ├── integration/
│   │   └── strategy-with-data.test.ts     # Strategy + real data
│   └── backtest/
│       └── rsi-backtest-regression.test.ts # Backtest results stability
└── vitest.config.ts
```

| Test Type | What It Validates | Required Coverage |
|-----------|------------------|-------------------|
| Unit | Strategy logic (signals, sizing) | 90%+ |
| Integration | Strategy + data pipeline | Key paths |
| Backtest Regression | Results don't change unexpectedly | Key metrics |

### Entry Criteria
- [ ] POC promoted with positive results
- [ ] GitHub issue created with acceptance criteria
- [ ] Historical data loaded in local TimescaleDB

### Exit Criteria (to promote to STAGING)
- [ ] Code follows strategy plugin interface
- [ ] Unit tests pass with > 90% coverage
- [ ] Integration tests pass
- [ ] Backtest regression baseline established
- [ ] Code reviewed and approved
- [ ] Documentation updated
- [ ] Merged to `develop` branch

---

## Phase 3: Staging / Paper Trading

### Goal
Validate that the strategy works with **live market data** and **simulated execution** — without risking real money.

### Environment
- **Where**: Staging Kubernetes cluster (K3s on dev server)
- **Data**: Live market data feeds (real WebSocket connections)
- **Execution**: Paper trading (simulated orders against live prices)
- **Monitoring**: Full Grafana dashboards

### What to Do

```
1. Deploy strategy to staging cluster
2. Connect to live data feeds
3. Run paper trading for 2-4 weeks
4. Monitor daily: PnL, drawdown, fill rates
5. Compare live results vs backtest expectations
6. Tune parameters if needed (document changes)
7. Final review with performance report
```

### Validation Criteria

| Metric | Backtest | Paper Trading | Tolerance |
|--------|----------|---------------|-----------|
| Sharpe Ratio | X | Y | Y > X × 0.7 |
| Win Rate | X% | Y% | Y > X - 10% |
| Max Drawdown | X% | Y% | Y < X × 1.5 |
| Avg Trade Duration | X | Y | Within 2x |
| Number of Trades | X/day | Y/day | Within 50% |

If paper trading results deviate more than tolerance from backtest, **investigate before promoting**.

### Common Issues at This Stage

| Issue | Likely Cause | Fix |
|-------|-------------|-----|
| Fewer trades than expected | Data feed gaps, timing differences | Check data pipeline, adjust timeframes |
| Higher slippage | Backtest assumed instant fills | Add realistic slippage model |
| Different PnL | Fees not modeled correctly | Update fee model |
| Strategy never triggers | Market regime changed | Validate on recent data, adjust parameters |

### Entry Criteria
- [ ] Feature branch merged to `develop`
- [ ] All tests passing in CI
- [ ] Staging environment healthy
- [ ] Live data feeds connected and validated

### Exit Criteria (to promote to PRODUCTION)
- [ ] 2+ weeks of paper trading completed
- [ ] Results within tolerance of backtest
- [ ] No critical bugs or data issues
- [ ] Risk manager rules working correctly
- [ ] Performance report reviewed and approved
- [ ] Kill switch tested and working

---

## Phase 4: Production

### Goal
Run the strategy with **real capital** under **strict risk management and monitoring**.

### Environment
- **Where**: Production Kubernetes cluster (EKS/GKE)
- **Data**: Live market data (redundant feeds)
- **Execution**: Real orders on exchanges
- **Monitoring**: Full observability + alerting + PagerDuty

### Deployment Process

```
1. Create release branch: release/v{X.Y.Z}
2. Run full test suite + backtest regression
3. Deploy to production (blue/green deployment)
4. Start with MINIMUM capital allocation (experimental tier)
5. Monitor closely for first 48 hours
6. Gradually increase allocation over 2 weeks if healthy
7. Move to standard capital tier after 4 weeks
```

### Capital Ramp-Up Schedule

| Week | Capital Allocation | Monitoring Level |
|------|-------------------|-----------------|
| Week 1 | 1% of portfolio (experimental) | Manual review 3x/day |
| Week 2 | 3% of portfolio | Manual review 1x/day |
| Week 3 | 5% of portfolio | Automated alerts |
| Week 4+ | Standard tier allocation | Automated alerts |

### Production Safeguards

```
┌─────────────────────────────────────────────────────┐
│                 Production Safeguards                │
│                                                      │
│  1. Per-trade risk limit      → max 2% of equity    │
│  2. Daily loss limit          → halt at -5%          │
│  3. Strategy drawdown limit   → pause at -15%        │
│  4. Global portfolio limit    → halt at -10%         │
│  5. Kill switch               → instant shutdown     │
│  6. Circuit breaker           → pause on API errors  │
│  7. Rate limiter              → respect exchange limits│
│  8. Duplicate order detection → prevent double fills  │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### Entry Criteria
- [ ] Paper trading results approved
- [ ] Production infrastructure healthy
- [ ] API keys configured (trade-only, no withdrawal)
- [ ] Monitoring dashboards operational
- [ ] Alert rules configured
- [ ] Kill switch tested
- [ ] Runbook documented for on-call

### Ongoing Operations
- [ ] Daily PnL review (automated report)
- [ ] Weekly strategy performance review
- [ ] Monthly parameter review and potential retraining
- [ ] Quarterly risk limit review
- [ ] API key rotation every 90 days

---

## Automated Promotion Pipeline

The goal is to automate as much of the promotion process as possible:

```yaml
# Conceptual promotion pipeline

promote_to_dev:
  trigger: Manual (PR from POC notebook)
  steps:
    - Create feature branch
    - Run linting + type checking
    - Run unit tests
    - Run integration tests
    - Run backtest regression
    - Auto-merge if all green + approved

promote_to_staging:
  trigger: Merge to develop
  steps:
    - Build Docker image
    - Push to container registry
    - Deploy to staging K3s
    - Start paper trading
    - Monitor for 2 weeks
    - Generate performance report
    - Notify for manual review

promote_to_production:
  trigger: Manual (approval required)
  steps:
    - Create release branch
    - Run full test suite
    - Build production Docker image
    - Blue/green deploy to production K8s
    - Health check (5 min)
    - If healthy → route traffic
    - If unhealthy → automatic rollback
    - Start capital ramp-up schedule
```

---

## Rollback & Kill Switch

### Automatic Rollback Triggers

| Trigger | Action | Recovery |
|---------|--------|---------|
| Health check fails on deploy | Roll back to previous version | Automatic |
| Strategy drawdown > 15% | Pause strategy, close positions | Manual review required |
| Daily portfolio loss > 10% | Halt ALL strategies | Manual review required |
| Exchange API errors > 10/min | Circuit breaker activates | Auto-retry after cooldown |
| Data feed down > 5 min | Pause affected strategies | Resume when feed recovers |

### Manual Kill Switch

```bash
# Emergency shutdown — closes all positions and halts all strategies
polytrade emergency-stop --close-positions

# Pause a specific strategy
polytrade strategy pause rsi_mean_reversion

# Resume after review
polytrade strategy resume rsi_mean_reversion --max-capital 1%
```

### Post-Incident Process

```
1. HALT    → Stop affected strategy/system
2. ASSESS  → What happened? How much was lost?
3. FIX     → Root cause analysis, implement fix
4. TEST    → Validate fix in staging
5. RESUME  → Restart with reduced capital
6. REVIEW  → Post-mortem document, update runbook
```

---

## Checklists

### New Strategy Checklist

```
POC Phase:
  □ Strategy template filled out
  □ Hypothesis documented
  □ Historical data obtained
  □ POC notebook with backtest
  □ Basic metrics meet thresholds
  □ Decision: promote / iterate / kill

Development Phase:
  □ GitHub issue created
  □ Feature branch created
  □ Strategy implements BaseStrategy interface
  □ Unit tests (>90% coverage)
  □ Integration tests
  □ Backtest regression baseline
  □ Code review approved
  □ Merged to develop

Staging Phase:
  □ Deployed to staging cluster
  □ Paper trading running 2+ weeks
  □ Results within tolerance of backtest
  □ Risk manager rules validated
  □ Kill switch tested
  □ Performance report approved

Production Phase:
  □ Release branch created
  □ Full test suite passing
  □ Blue/green deployment successful
  □ Capital at experimental tier (1%)
  □ Monitoring and alerts active
  □ Runbook documented
  □ 48-hour close monitoring complete
  □ Gradual ramp-up initiated
```

### Production Readiness Checklist

```
Infrastructure:
  □ K8s cluster healthy
  □ Database backed up
  □ Redis cluster healthy
  □ Monitoring stack operational
  □ Alert routes configured

Security:
  □ API keys are trade-only (no withdrawals)
  □ Keys stored in Vault/K8s Secrets
  □ Network policies applied
  □ IP whitelisting configured

Operations:
  □ Runbook documented
  □ On-call rotation set up
  □ Escalation path defined
  □ Post-mortem template ready
```
