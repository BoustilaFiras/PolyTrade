# PolyTrade

**AI-powered automated trading system for crypto, prediction markets, and beyond.**

PolyTrade is an open-source algorithmic trading platform that combines machine learning, real-time market data, and robust infrastructure to execute trading strategies across multiple markets — including cryptocurrency exchanges and prediction markets like Polymarket.

---

## Vision

Build a modular, production-grade trading system that:

- **Analyzes** markets using AI/ML models and technical indicators
- **Executes** trades automatically across crypto exchanges and prediction markets
- **Backtests** strategies on historical data before deploying real capital
- **Scales** with cloud-native infrastructure (Kubernetes, Ansible, CI/CD)
- **Learns** and adapts through continuous model retraining and feedback loops

## Target Markets

| Market | Platform | Type |
|--------|----------|------|
| Cryptocurrency | Binance, Bybit, Kraken | Spot & Futures |
| Prediction Markets | Polymarket | Binary/Multi-outcome events |
| DeFi | Uniswap, Aave | Swaps, Lending, Yield |

## Project Status

> **Phase: Documentation & Design** — We are currently in the planning phase, defining strategies, architecture, and infrastructure before writing any trading code.

## Documentation

| Document | Description |
|----------|-------------|
| [Strategy Catalog](docs/strategies/README.md) | Trading strategies, how to collect them, evaluation framework |
| [Architecture](docs/architecture/README.md) | System design, component diagrams, pros/cons analysis |
| [Infrastructure](docs/infrastructure/README.md) | Tech stack, DevOps tooling, cloud setup |
| [Pipeline](docs/pipeline/README.md) | POC → Development → Production lifecycle |

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        PolyTrade Platform                       │
├──────────────────────────────┬──────────────────────────────────┤
│    Node.js / TypeScript      │        Python Workers            │
│  ┌──────────┐ ┌──────────┐  │  ┌──────────┐ ┌──────────────┐  │
│  │  Data    │ │ Strategy │  │  │    AI    │ │  Backtesting │  │
│  │Collector │ │  Engine  │  │  │  Models  │ │    Engine    │  │
│  ├──────────┤ ├──────────┤  │  └──────────┘ └──────────────┘  │
│  │   Risk   │ │Execution │  │                                  │
│  │ Manager  │ │  Engine  │  │                                  │
│  └──────────┘ └──────────┘  │                                  │
├──────────────────────────────┴──────────────────────────────────┤
│                   Message Bus (Redis Streams)                    │
├─────────────────────────────────────────────────────────────────┤
│              Infrastructure (K8s / Docker / Cloud)               │
└─────────────────────────────────────────────────────────────────┘
```

## Tech Stack

**Hybrid architecture**: Node.js/TypeScript for server services + Python for AI/ML workers.

| Layer | Technology |
|-------|-----------|
| Server | TypeScript + Node.js 22 LTS |
| API | Fastify |
| Real-time | WebSocket (ws) |
| AI/ML Workers | Python 3.12+ (PyTorch, scikit-learn, LangChain) |
| Backtesting | Python (vectorbt, custom engine) |
| Data | pandas, NumPy, TimescaleDB |
| Message Bus | Redis Streams / Apache Kafka |
| Containers | Docker |
| Orchestration | Kubernetes (K3s for dev, EKS/GKE for prod) |
| IaC | Ansible + Terraform |
| CI/CD | GitHub Actions |
| Monitoring | Grafana + Prometheus |
| Monorepo | Turborepo + pnpm |
| Cloud | AWS / GCP |

## Getting Started

> Coming soon — project is in documentation phase.

## Contributing

We welcome contributors! Check the [GitHub Project Board](../../projects) for open tasks and the [Contributing Guide](docs/CONTRIBUTING.md) for how to get involved.

## License

MIT License — see [LICENSE](LICENSE) for details.


