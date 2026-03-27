# Infrastructure & Tech Stack

This document details every technology in the PolyTrade stack, the DevOps ecosystem, cloud architecture, and how everything fits together.

**Key design decision**: We use a **hybrid stack** — Node.js/TypeScript for all server-side services (fast, scalable, great for I/O-heavy work like WebSockets and API calls) and Python only for AI/ML workers and backtesting (where its data science ecosystem is unmatched).

---

## Table of Contents

1. [Tech Stack Overview](#tech-stack-overview)
2. [Why Hybrid Node.js + Python](#why-hybrid-nodejs--python)
3. [Development Environment](#development-environment)
4. [DevOps Ecosystem](#devops-ecosystem)
5. [Cloud Architecture](#cloud-architecture)
6. [CI/CD Pipeline](#cicd-pipeline)
7. [Monitoring & Alerting](#monitoring--alerting)
8. [Cost Estimation](#cost-estimation)

---

## Tech Stack Overview

### Server Layer (Node.js / TypeScript)

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Language | TypeScript 5+ | Type-safe server code |
| Runtime | Node.js 22 LTS | Server runtime |
| Package Manager | pnpm | Fast, disk-efficient dependencies |
| Web Framework | Fastify | REST API (faster than Express) |
| WebSocket | ws / Socket.io | Real-time data feeds, live updates |
| Task Queue | BullMQ (Redis-backed) | Job processing, scheduled tasks |
| ORM | Drizzle ORM / Prisma | Database abstraction |
| Validation | Zod | Runtime type validation |
| Monorepo | Turborepo | Manage multiple packages |

### AI/ML Layer (Python)

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Language | Python 3.12+ | ML/AI workloads |
| Package Manager | uv | Fast dependency management |
| ML Framework | scikit-learn, PyTorch | Model training |
| Experiment Tracking | MLflow | Model versioning, metrics |
| LLM Integration | LangChain + Claude API | Sentiment analysis, event parsing |
| Data Processing | pandas, NumPy | Feature engineering |
| Backtesting | vectorbt / custom | Strategy backtesting |
| Visualization | matplotlib, Plotly | Backtest reports |

### Data Layer

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Time-Series DB | TimescaleDB (PostgreSQL) | OHLCV data, tick data, order history |
| Cache | Redis | Real-time prices, session data, pub/sub |
| Message Bus | Redis Streams → Kafka | Inter-service communication |
| Object Storage | MinIO / S3 | ML model artifacts, backtest reports |

### Infrastructure

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Containers | Docker | Package and isolate services |
| Orchestration | K3s (dev) / EKS (prod) | Container management |
| IaC | Terraform | Cloud resource provisioning |
| Config Management | Ansible | Server setup, app deployment |
| CI/CD | GitHub Actions | Build, test, deploy |
| Container Registry | GitHub Container Registry | Docker image storage |
| Secrets | HashiCorp Vault / K8s Secrets | API keys, credentials |

### Monitoring

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Metrics | Prometheus | System and business metrics |
| Logs | Loki | Centralized log aggregation |
| Traces | Jaeger / OpenTelemetry | Distributed tracing |
| Dashboards | Grafana | Visualization of all above |
| Alerting | Grafana Alerting | Incident notification |

---

## Why Hybrid Node.js + Python

### The Split

```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│   Node.js / TypeScript                Python                 │
│   ════════════════════                ══════                  │
│                                                              │
│   ✅ API Gateway                     ✅ ML Model Training    │
│   ✅ Data Collectors (WebSocket)     ✅ Inference Server     │
│   ✅ Execution Engine                ✅ Backtesting Engine   │
│   ✅ Risk Manager                    ✅ Feature Engineering  │
│   ✅ Order Management                ✅ Jupyter Notebooks    │
│   ✅ Portfolio Tracker               ✅ Sentiment Analysis   │
│   ✅ Dashboard Backend               ✅ Data Science POCs    │
│   ✅ Real-time Event Processing                              │
│                                                              │
│   Communication: Redis Streams / gRPC / HTTP                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Why Node.js for Server

| Advantage | Details |
|-----------|---------|
| **Async I/O** | Event loop handles thousands of concurrent WebSocket connections natively |
| **Speed** | V8 engine is fast for I/O-bound work; no GIL problem |
| **Scaling** | Worker threads + cluster mode for CPU-bound tasks |
| **TypeScript** | Strong type safety, better refactoring, catch bugs at compile time |
| **Ecosystem** | npm has libraries for every exchange (ccxt, binance, etc.) |
| **Real-time** | WebSocket handling is first-class, perfect for market data |
| **Familiar** | Matches our professional experience → faster development |

### Why Python for AI/ML Only

| Advantage | Details |
|-----------|---------|
| **ML Ecosystem** | PyTorch, scikit-learn, pandas — nothing comparable in JS |
| **Backtesting** | vectorbt, backtrader — mature and battle-tested |
| **Research** | Jupyter notebooks for rapid experimentation |
| **LLM Tools** | LangChain, Claude SDK — most AI tools are Python-first |

### How They Communicate

```
Node.js Services ◄──── Redis Streams ────► Python Workers
                 ◄──── gRPC ──────────────►
                 ◄──── HTTP/REST ─────────►

Example flow:
1. Node.js Data Collector receives market data via WebSocket
2. Publishes to Redis Streams
3. Python ML Worker consumes, runs inference
4. Publishes prediction back to Redis Streams
5. Node.js Strategy Engine consumes prediction
6. Node.js Execution Engine places order
```

---

## Development Environment

### Project Structure (Monorepo with Turborepo)

```
PolyTrade/
├── apps/                          # Deployable services
│   ├── api-gateway/               # Fastify REST API (TS)
│   ├── data-collector/            # Market data ingestion (TS)
│   ├── strategy-engine/           # Strategy execution (TS)
│   ├── execution-engine/          # Order management (TS)
│   ├── risk-manager/              # Risk validation (TS)
│   └── dashboard/                 # Web UI (React/Next.js)
│
├── workers/                       # Python workers
│   ├── ml-inference/              # ML model serving
│   ├── ml-training/               # Model training pipelines
│   ├── backtester/                # Backtesting engine
│   └── sentiment/                 # News/social sentiment
│
├── packages/                      # Shared TypeScript packages
│   ├── shared-types/              # Shared TypeScript types
│   ├── exchange-adapters/         # Binance, Polymarket connectors
│   ├── indicators/                # Technical indicators (ta.js)
│   └── risk-rules/                # Risk management logic
│
├── infra/
│   ├── docker/                    # Dockerfiles per service
│   ├── k8s/                       # Kubernetes manifests
│   ├── ansible/                   # Provisioning playbooks
│   └── terraform/                 # Cloud infrastructure
│
├── docs/                          # Documentation
├── tests/                         # E2E tests
├── .github/workflows/             # CI/CD
├── turbo.json                     # Turborepo config
├── pnpm-workspace.yaml            # pnpm workspace
└── docker-compose.yml             # Local dev stack
```

### Local Setup

```
Developer Machine
│
├── Docker Compose (local infrastructure)
│   ├── timescaledb           (database)
│   ├── redis                 (cache + message bus)
│   ├── grafana               (dashboards)
│   └── prometheus            (metrics)
│
├── Node.js Services (hot-reload via tsx/nodemon)
│   ├── api-gateway
│   ├── data-collector
│   ├── strategy-engine
│   └── execution-engine
│
├── Python Workers (virtual env via uv)
│   ├── ml-inference
│   └── backtester
│
└── Dev Tools
    ├── VS Code (recommended)
    ├── DBeaver (database GUI)
    └── Postman / httpie (API testing)
```

### Dev Environment Requirements

```yaml
# Minimum
CPU: 4 cores
RAM: 8 GB
Disk: 20 GB SSD
Node.js: 22 LTS
Python: 3.12+
Docker: 24+

# Recommended
CPU: 8 cores
RAM: 16 GB
Disk: 50 GB SSD
GPU: Optional (for ML training)
```

### Setup Commands (planned)

```bash
# Clone and setup
git clone https://github.com/BoustilaFiras/PolyTrade.git
cd PolyTrade

# Install Node.js dependencies
pnpm install

# Install Python dependencies
cd workers && uv sync && cd ..

# Start infrastructure
docker compose up -d

# Start all services (dev mode)
pnpm dev

# Run tests
pnpm test

# Run Python backtester
cd workers/backtester && uv run backtest --strategy rsi
```

---

## DevOps Ecosystem

### Ansible — Configuration Management

```
infra/ansible/
├── inventory/
│   ├── dev.yml
│   ├── staging.yml
│   └── production.yml
├── playbooks/
│   ├── setup-server.yml         # Base server (users, packages, firewall)
│   ├── deploy-app.yml           # Deploy services
│   ├── setup-monitoring.yml     # Grafana/Prometheus
│   └── rotate-secrets.yml       # API key rotation
├── roles/
│   ├── common/                  # Base packages, hardening
│   ├── docker/                  # Docker installation
│   ├── k3s/                     # K3s cluster setup
│   ├── nodejs/                  # Node.js runtime
│   ├── python/                  # Python runtime (for workers)
│   ├── monitoring/              # Prometheus + Grafana
│   └── app/                     # PolyTrade services
└── ansible.cfg
```

### Terraform — Infrastructure as Code

```
infra/terraform/
├── modules/
│   ├── networking/              # VPC, subnets, security groups
│   ├── compute/                 # EC2/GCE instances
│   ├── database/                # RDS (TimescaleDB)
│   ├── kubernetes/              # EKS/GKE cluster
│   ├── cache/                   # ElastiCache (Redis)
│   └── storage/                 # S3 buckets
├── environments/
│   ├── dev/
│   ├── staging/
│   └── production/
└── backend.tf                   # Remote state
```

### Kubernetes — Orchestration

```
infra/k8s/
├── base/
│   ├── namespace.yml
│   ├── configmap.yml
│   └── secrets.yml
├── apps/
│   ├── data-collector/
│   │   ├── deployment.yml
│   │   ├── service.yml
│   │   └── hpa.yml              # Auto-scaling
│   ├── strategy-engine/
│   ├── execution-engine/
│   ├── risk-manager/
│   └── api-gateway/
├── workers/
│   ├── ml-inference/
│   └── backtester/
├── monitoring/
│   ├── prometheus/
│   ├── grafana/
│   └── loki/
└── kustomization.yml
```

### Docker — Containerization

```
infra/docker/
├── Dockerfile.node-base         # Shared Node.js base
├── Dockerfile.python-base       # Shared Python base
├── Dockerfile.data-collector    # Node.js
├── Dockerfile.strategy-engine   # Node.js
├── Dockerfile.execution-engine  # Node.js
├── Dockerfile.ml-inference      # Python
├── Dockerfile.backtester        # Python
├── docker-compose.yml           # Local dev
├── docker-compose.test.yml      # CI testing
└── .dockerignore
```

---

## Cloud Architecture

### AWS Architecture

```
┌─────────────────────────────────────────────────────────┐
│                        AWS Cloud                         │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │                    VPC                            │    │
│  │                                                   │    │
│  │  ┌──────────┐  ┌──────────────────────────────┐  │    │
│  │  │  ALB     │  │   EKS Cluster                │  │    │
│  │  │(ingress) │─▶│                              │  │    │
│  │  └──────────┘  │  Node.js        Python       │  │    │
│  │                │  ┌──────┐      ┌──────────┐  │  │    │
│  │                │  │ Data │      │ML Worker │  │  │    │
│  │                │  │Collect│      │(inference)│  │  │    │
│  │                │  ├──────┤      ├──────────┤  │  │    │
│  │                │  │Strat │      │Backtester│  │  │    │
│  │                │  │Engine│      └──────────┘  │  │    │
│  │                │  ├──────┤                    │  │    │
│  │                │  │ Exec │                    │  │    │
│  │                │  │Engine│                    │  │    │
│  │                │  └──────┘                    │  │    │
│  │                └──────────────────────────────┘  │    │
│  │                                                   │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │    │
│  │  │   RDS    │  │ElastiCache│  │     S3       │   │    │
│  │  │(Postgres)│  │ (Redis)   │  │(ML artifacts)│   │    │
│  │  └──────────┘  └──────────┘  └──────────────┘   │    │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐      │
│  │CloudWatch│  │  ECR     │  │  Secrets Manager  │      │
│  └──────────┘  └──────────┘  └──────────────────┘      │
└─────────────────────────────────────────────────────────┘
```

### Cost-Optimized Dev Setup

| Resource | Cheap Option | Monthly Cost |
|----------|-------------|-------------|
| Server | Hetzner CX31 (4 vCPU, 8GB) | ~$8 |
| Database | PostgreSQL on same server | $0 |
| K3s | On same server | $0 |
| Domain | Optional | ~$1 |
| **Total** | | **~$9/month** |

---

## CI/CD Pipeline

### Pipeline Overview

```
┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐
│ Push │───▶│ Lint │───▶│ Test │───▶│Build │───▶│Deploy│───▶│Verify│
│      │    │      │    │      │    │Image │    │      │    │      │
└──────┘    └──────┘    └──────┘    └──────┘    └──────┘    └──────┘
                                                   │
                                        ┌──────────┼──────────┐
                                        ▼          ▼          ▼
                                      Dev      Staging    Production
                                    (auto)     (auto)    (manual gate)
```

### GitHub Actions Workflows

```yaml
# .github/workflows/ci.yml — runs on every PR
- TypeScript: eslint, tsc --noEmit (type check)
- Python: ruff, mypy
- Node.js unit tests (vitest)
- Python unit tests (pytest)
- Integration tests (with Docker services)
- Build Docker images
- Push to GitHub Container Registry

# .github/workflows/deploy-dev.yml — on merge to main
- Auto-deploy to dev

# .github/workflows/deploy-staging.yml — on release branch
- Deploy to staging + smoke tests + backtest regression

# .github/workflows/deploy-prod.yml — manual trigger
- Requires approval → blue/green deploy → rollback on failure
```

### Branch Strategy

```
main (stable)
├── develop (integration)
│   ├── feature/strategy-rsi
│   ├── feature/binance-connector
│   └── fix/order-execution-bug
├── release/v1.0.0 (staging)
└── hotfix/critical-bug (emergency)
```

---

## Monitoring & Alerting

### Key Metrics

| Category | Metric | Alert Threshold |
|----------|--------|----------------|
| **Trading** | Portfolio PnL | Daily loss > 5% |
| **Trading** | Strategy drawdown | > 15% |
| **Trading** | Order fill rate | < 90% |
| **Trading** | Slippage | > 0.5% |
| **System** | API latency (exchange) | > 2s |
| **System** | CPU usage | > 80% sustained |
| **System** | Memory usage | > 85% |
| **System** | Event loop lag (Node.js) | > 100ms |
| **Data** | WebSocket reconnects | > 3 in 5 min |
| **Data** | Data feed lag | > 5s |

### Grafana Dashboards (Planned)

1. **Portfolio Overview** — Real-time PnL, allocation, exposure
2. **Strategy Performance** — Per-strategy returns, Sharpe, drawdown
3. **Execution Quality** — Fill rates, slippage, latency
4. **System Health** — CPU, memory, event loop, pod status
5. **Data Pipeline** — Feed health, WebSocket status, gaps

---

## Cost Estimation

### Development Phase

| Resource | Provider | Monthly Cost |
|----------|---------|-------------|
| VPS (dev/staging) | Hetzner | $8 |
| Domain | Namecheap | $1 |
| GitHub (public repo) | GitHub | $0 |
| Exchange API | Binance/Polymarket | $0 |
| **Total** | | **~$9/month** |

### Production Phase

| Resource | Provider | Monthly Cost |
|----------|---------|-------------|
| EKS cluster (3 nodes) | AWS | $150 |
| RDS (TimescaleDB) | AWS | $30 |
| ElastiCache (Redis) | AWS | $15 |
| S3 | AWS | $5 |
| ALB | AWS | $20 |
| Monitoring | Self-hosted | $0 |
| **Total** | | **~$220/month** |

### Cost Optimization Tips

- Use spot instances for backtest workloads (70% savings)
- Schedule dev/staging to shut down at night
- Self-host monitoring stack
- Start with single-node K3s, scale when needed
