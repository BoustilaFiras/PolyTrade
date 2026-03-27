# System Architecture

This document describes the PolyTrade system architecture, explains design decisions with their trade-offs, and provides component diagrams for each major subsystem.

---

## Table of Contents

1. [Design Principles](#design-principles)
2. [High-Level Architecture](#high-level-architecture)
3. [Component Deep Dive](#component-deep-dive)
4. [Architecture Propositions](#architecture-propositions)
5. [Data Flow](#data-flow)
6. [Technology Choices — Pros & Cons](#technology-choices--pros--cons)
7. [Security Architecture](#security-architecture)

---

## Design Principles

| Principle | Description |
|-----------|-------------|
| **Modularity** | Each component is independently deployable and replaceable |
| **Event-Driven** | Components communicate through events, not direct calls |
| **Fail-Safe** | System defaults to safe state (no trades) on any failure |
| **Observable** | Every component emits metrics, logs, and traces |
| **Backtestable** | Production code and backtest code share the same strategy logic |
| **Market-Agnostic** | Core engine doesn't know about specific markets — adapters do |

---

## High-Level Architecture

```
                              ┌─────────────────┐
                              │   GitHub Repo    │
                              │  (Source of Truth)│
                              └────────┬─────────┘
                                       │ CI/CD
                              ┌────────▼─────────┐
                              │  GitHub Actions   │
                              │  Build/Test/Deploy│
                              └────────┬─────────┘
                                       │
                    ┌──────────────────▼──────────────────┐
                    │         Kubernetes Cluster           │
                    │                                      │
                    │  ┌──────────┐    ┌──────────────┐   │
                    │  │  Data    │    │   Strategy    │   │
                    │  │Collectors│───▶│    Engine     │   │
                    │  │(pods)    │    │   (pods)      │   │
                    │  └──────────┘    └──────┬───────┘   │
                    │       │                  │           │
                    │       ▼                  ▼           │
                    │  ┌──────────┐    ┌──────────────┐   │
                    │  │TimescaleDB│   │  Risk Manager │   │
                    │  │(stateful) │   │   (pod)       │   │
                    │  └──────────┘    └──────┬───────┘   │
                    │                         │           │
                    │                  ┌──────▼───────┐   │
                    │                  │  Execution    │   │
                    │                  │  Engine (pod) │   │
                    │                  └──────┬───────┘   │
                    │                         │           │
                    │  ┌──────────┐    ┌──────▼───────┐   │
                    │  │ Grafana  │◀───│  Redis/Kafka  │   │
                    │  │Prometheus│    │ (message bus) │   │
                    │  └──────────┘    └──────────────┘   │
                    │                                      │
                    └──────────────────────────────────────┘
                                       │
                              ┌────────▼─────────┐
                              │ Exchange APIs     │
                              │ Binance/Polymarket│
                              └──────────────────┘
```

---

## Component Deep Dive

### 1. Data Collectors

**Responsibility**: Ingest real-time and historical market data from multiple sources.

```
┌─────────────────────────────────────────────────┐
│                Data Collectors                    │
│                                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐│
│  │  Binance WS  │  │ Polymarket  │  │  News    ││
│  │  Connector   │  │ CLOB API    │  │  Feeds   ││
│  └──────┬──────┘  └──────┬──────┘  └────┬─────┘│
│         │                │               │       │
│         ▼                ▼               ▼       │
│  ┌─────────────────────────────────────────────┐│
│  │          Unified Data Normalizer             ││
│  │  (converts all feeds to standard schema)     ││
│  └──────────────────┬──────────────────────────┘│
│                     │                            │
│         ┌───────────┼───────────┐                │
│         ▼           ▼           ▼                │
│    TimescaleDB   Redis       Message Bus         │
│    (storage)     (cache)     (events)            │
└─────────────────────────────────────────────────┘
```

**Key decisions**:
- Each exchange gets its own connector pod → can scale/restart independently
- All data normalized to a common schema before storage
- WebSocket preferred over REST for real-time data (lower latency)

### 2. Strategy Engine

**Responsibility**: Run trading strategies, generate buy/sell signals.

```
┌─────────────────────────────────────────────────┐
│                Strategy Engine                    │
│                                                   │
│  ┌─────────────────────────────────────────────┐│
│  │            Strategy Registry                  ││
│  │  (discovers and loads strategy plugins)       ││
│  └──────────────────┬──────────────────────────┘│
│                     │                            │
│  ┌──────────┐  ┌────▼─────┐  ┌──────────────┐  │
│  │ TA-based │  │ ML-based │  │ Prediction   │  │
│  │Strategies│  │Strategies│  │ Market Strats│  │
│  └────┬─────┘  └────┬─────┘  └──────┬───────┘  │
│       │              │               │           │
│       ▼              ▼               ▼           │
│  ┌─────────────────────────────────────────────┐│
│  │           Signal Aggregator                   ││
│  │  (combines signals, resolves conflicts)       ││
│  └──────────────────┬──────────────────────────┘│
│                     │                            │
│                     ▼                            │
│              Signal Event → Message Bus          │
└─────────────────────────────────────────────────┘
```

**Key decisions**:
- Plugin architecture → add new strategies without touching core
- Same strategy code runs in backtest and live mode
- Signal aggregator handles conflicting signals from multiple strategies

### 3. Risk Manager

**Responsibility**: Validate signals against risk rules before execution.

```
Signal In → [Position Sizing] → [Exposure Check] → [Drawdown Check] → [Correlation Check] → Approved/Rejected
```

**Rules enforced**:
- Per-trade risk limits (max 2% portfolio)
- Per-strategy drawdown limits
- Global portfolio exposure limits
- Correlation limits between active positions
- Kill switch capability

### 4. Execution Engine

**Responsibility**: Convert approved signals into actual orders on exchanges.

```
┌─────────────────────────────────────────────────┐
│              Execution Engine                     │
│                                                   │
│  ┌────────────┐                                  │
│  │  Order     │  Approved Signal                 │
│  │  Builder   │◀──────────────                   │
│  └─────┬──────┘                                  │
│        │                                         │
│        ▼                                         │
│  ┌────────────┐    ┌─────────────┐              │
│  │  Smart     │    │  Exchange   │              │
│  │  Router    │───▶│  Adapters   │              │
│  │(best exec) │    │             │              │
│  └────────────┘    │ ┌─────────┐ │              │
│                    │ │Binance  │ │              │
│  ┌────────────┐    │ ├─────────┤ │              │
│  │  Order     │◀───│ │Polymar. │ │              │
│  │  Tracker   │    │ ├─────────┤ │              │
│  └────────────┘    │ │Bybit    │ │              │
│                    │ └─────────┘ │              │
│                    └─────────────┘              │
└─────────────────────────────────────────────────┘
```

### 5. AI/ML Service

**Responsibility**: Train and serve ML models for strategy signals.

```
┌──────────────────────────────────────────────────┐
│                 AI/ML Service                      │
│                                                    │
│  ┌──────────────┐   ┌──────────────────────────┐ │
│  │  Model Store  │   │   Training Pipeline       │ │
│  │  (MLflow)     │◀──│   (scheduled retraining)  │ │
│  └──────┬───────┘   └──────────────────────────┘ │
│         │                                         │
│         ▼                                         │
│  ┌──────────────┐   ┌──────────────────────────┐ │
│  │  Inference    │   │  Feature Store            │ │
│  │  Server       │◀──│  (pre-computed features)  │ │
│  │  (FastAPI)    │   └──────────────────────────┘ │
│  └──────┬───────┘                                 │
│         │                                         │
│         ▼                                         │
│    Prediction → Strategy Engine                   │
└──────────────────────────────────────────────────┘
```

### 6. Monitoring & Observability

```
┌─────────────────────────────────────────────┐
│            Observability Stack                │
│                                               │
│  ┌───────────┐  ┌──────────┐  ┌───────────┐ │
│  │Prometheus  │  │  Loki    │  │  Jaeger   │ │
│  │(metrics)   │  │ (logs)   │  │ (traces)  │ │
│  └─────┬─────┘  └────┬─────┘  └─────┬─────┘ │
│        │              │              │        │
│        └──────────────┼──────────────┘        │
│                       ▼                       │
│              ┌──────────────┐                 │
│              │   Grafana    │                 │
│              │  Dashboards  │                 │
│              └──────────────┘                 │
│                                               │
│  Key Dashboards:                              │
│  - Portfolio PnL (real-time)                  │
│  - Strategy performance comparison            │
│  - System health (latency, errors, uptime)    │
│  - Risk exposure heatmap                      │
│  - Alert: drawdown, anomaly, system failure   │
└─────────────────────────────────────────────┘
```

---

## Architecture Propositions

### Proposition A: Monolith-First (Recommended for POC)

```
Single Node.js process
├── Data collection (async WebSocket)
├── Strategy engine (event loop)
├── Risk checks (inline)
├── Execution (async API calls)
└── SQLite/PostgreSQL (local)
+ Python subprocess for ML inference (optional)
```

**Pros**:
- Fast to build and iterate
- Easy to debug (single process, single debugger)
- No network overhead between components
- Simple deployment (one container)

**Cons**:
- Doesn't scale horizontally
- Single point of failure
- Hard to deploy strategies independently
- Memory-bound by single machine

**Best for**: POC phase, single strategy, learning

### Proposition B: Microservices (Recommended for Production)

```
Kubernetes cluster
├── Node.js pods:
│   ├── Data collector (per exchange)
│   ├── Strategy engine (per strategy)
│   ├── Risk manager
│   └── Execution engine
├── Python pods:
│   ├── ML inference worker
│   └── Backtesting worker
├── Message bus (Redis Streams / Kafka)
├── Database (TimescaleDB)
└── Monitoring stack (Grafana/Prometheus)
```

**Pros**:
- Each component scales independently
- Fault isolation (one strategy crash doesn't kill others)
- Independent deployment of strategies
- Team members can work on different services

**Cons**:
- Higher operational complexity
- Network latency between services
- Harder to debug distributed issues
- More infrastructure to manage

**Best for**: Production, multiple strategies, team collaboration

### Proposition C: Hybrid (Recommended Path)

```
Phase 1 (POC):      Monolith → validate strategies work
Phase 2 (Dev):      Extract data + execution into services, keep strategies together
Phase 3 (Prod):     Full microservices with K8s
```

**Pros**:
- Start simple, grow complexity as needed
- Learn infrastructure incrementally
- Each phase delivers value
- Don't over-engineer before validating the core trading logic

**Cons**:
- Requires refactoring between phases
- Must design interfaces early to make extraction possible

**This is our recommended approach.**

---

## Data Flow

### Real-Time Trading Flow

```
Exchange WebSocket
    │
    ▼
Data Collector ──persist──▶ TimescaleDB
    │
    │ publish(market_data_event)
    ▼
Message Bus (Redis Streams)
    │
    ▼
Strategy Engine
    │ consume(market_data_event)
    │ compute signals
    │ publish(signal_event)
    ▼
Message Bus
    │
    ▼
Risk Manager
    │ validate(signal_event)
    │ publish(approved_order_event)
    ▼
Message Bus
    │
    ▼
Execution Engine
    │ execute(approved_order_event)
    │ publish(fill_event)
    ▼
Message Bus ──▶ Portfolio Tracker ──▶ Grafana Dashboard
```

### Backtesting Flow

```
TimescaleDB (historical data)
    │
    ▼
Backtest Engine
    │ replay market data chronologically
    │ feed to Strategy Engine (same code as live)
    │ simulate execution (slippage, fees)
    │
    ▼
Backtest Results
    │
    ▼
Performance Report (Sharpe, drawdown, PnL curve)
```

---

## Technology Choices — Pros & Cons

### Server Language: TypeScript / Node.js

| Pros | Cons |
|------|------|
| Excellent async I/O (event loop) — perfect for WebSockets | Not ideal for CPU-heavy computation |
| TypeScript gives strong type safety at compile time | Smaller finance-specific library ecosystem than Python |
| npm has exchange libraries (ccxt, etc.) | Less ML/AI tooling |
| Scales well with worker threads + clustering | |
| Familiar from professional experience | |

**Verdict**: Right choice for server-side services. The event loop handles concurrent WebSocket feeds and API calls naturally. TypeScript catches bugs early.

### AI/ML Language: Python

| Pros | Cons |
|------|------|
| Best ML/AI library support (PyTorch, scikit-learn) | GIL limits true parallelism |
| Rich finance ecosystem (pandas, numpy, vectorbt) | Slower for server workloads |
| Jupyter notebooks for rapid experimentation | |
| LLM tools are Python-first (LangChain, Claude SDK) | |

**Verdict**: Used strictly for ML workers and backtesting — where its ecosystem is unmatched. Not for server-side services.

### Database: TimescaleDB vs InfluxDB vs QuestDB

| Feature | TimescaleDB | InfluxDB | QuestDB |
|---------|------------|----------|---------|
| SQL support | Full PostgreSQL | InfluxQL/Flux | SQL-like |
| Ecosystem | PostgreSQL extensions | Limited | Growing |
| Performance | Good | Good for writes | Excellent |
| Learning curve | Low (it's Postgres) | Medium | Medium |
| Community | Large | Large | Small |

**Verdict**: TimescaleDB — it's PostgreSQL with time-series superpowers. We get SQL, extensions, and a massive ecosystem.

### Message Bus: Redis Streams vs Apache Kafka

| Feature | Redis Streams | Apache Kafka |
|---------|--------------|-------------|
| Latency | Sub-millisecond | Low milliseconds |
| Throughput | Moderate | Very high |
| Persistence | Optional | Built-in |
| Complexity | Low | High |
| Resource usage | Light | Heavy (JVM, ZooKeeper) |

**Verdict**: Start with Redis Streams (simpler, lighter). Migrate to Kafka if we need higher throughput or stronger persistence guarantees at scale.

### Orchestration: K3s vs K8s vs Docker Compose

| Feature | Docker Compose | K3s | Full K8s (EKS/GKE) |
|---------|---------------|-----|-------------------|
| Complexity | Very low | Low | High |
| Production-ready | No | Yes (small scale) | Yes |
| Auto-scaling | No | Yes | Yes |
| Cost | Free | Free (self-hosted) | $$$ |
| Learning curve | Easy | Medium | Steep |

**Verdict**: Docker Compose for local dev → K3s for staging → EKS/GKE for production.

---

## Security Architecture

### API Key Management

```
┌─────────────────────────────────────┐
│   Secrets Management                 │
│                                      │
│   Dev:  .env files (git-ignored)     │
│   Prod: K8s Secrets + Vault          │
│                                      │
│   NEVER:                             │
│   - Hardcode API keys in source      │
│   - Commit .env files                │
│   - Share keys in plaintext          │
│   - Use production keys in dev       │
└─────────────────────────────────────┘
```

### Network Security

- All exchange API calls over HTTPS
- Internal service communication over mTLS (in K8s)
- IP whitelisting on exchange API keys where supported
- Rate limiting on all outbound API calls

### Access Control

- Read-only API keys for data collection
- Trade-only keys with withdrawal disabled for execution
- Separate keys per environment (dev/staging/prod)
- Key rotation every 90 days
