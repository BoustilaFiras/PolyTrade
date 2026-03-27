# Contributing to PolyTrade

Thanks for your interest in contributing! This guide will help you get started.

## How to Contribute

### 1. Find Something to Work On

- Check the [GitHub Project Board](https://github.com/BoustilaFiras/PolyTrade/projects) for open tasks
- Look for issues labeled `good first issue` or `help wanted`
- Have an idea? Open an issue first to discuss it

### 2. Fork and Branch

```bash
git clone https://github.com/YOUR_USERNAME/PolyTrade.git
cd PolyTrade
git checkout -b feature/your-feature-name
```

### 3. Follow the Branch Naming Convention

- `feature/` — new features
- `fix/` — bug fixes
- `docs/` — documentation changes
- `infra/` — infrastructure changes
- `refactor/` — code refactoring

### 4. Make Your Changes

- Follow existing code style
- TypeScript for server services, Python for ML workers
- Write tests for new functionality
- Update documentation if needed

### 5. Submit a Pull Request

- Write a clear PR description
- Reference related issues
- Ensure CI passes
- Request review

## Code Standards

### TypeScript (Server Services)

- Use strict TypeScript (`strict: true`)
- Use ESLint + Prettier
- Write tests with Vitest
- Use Zod for runtime validation

### Python (ML Workers)

- Use type hints
- Use ruff for linting
- Write tests with pytest
- Use uv for dependency management

## Questions?

Open an issue or start a discussion on GitHub.
