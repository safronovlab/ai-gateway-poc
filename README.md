# AI Gateway

**Async API gateway between business logic and LLM providers.**

A production-ready intelligent proxy layer between internal services and external LLM providers (via Portkey). Centralized request logging, runtime guardrail policies, encrypted credential storage, and a web admin UI. Built on Clean Architecture with full async I/O end to end.

---

## What it does

| Capability | Detail |
|---|---|
| LLM proxy | Routes requests from internal services to external providers through Portkey with virtual key support |
| Guardrails | Runtime policies with cloud sync, enable / disable per request, HTTP 446 blocked response |
| Credential security | Provider API keys encrypted at rest with Fernet, never stored in plaintext |
| Audit and observability | Per-request log: provider, model, latency, status, blocked-by-guardrail flag |
| Admin UI | Manage providers, policies, browse logs, view aggregate statistics |
| Webhook ingress | Signature verification via `X-Webhook-Secret` for inbound integrations |

---

## Architecture

Clean Architecture with four isolated layers. Each inner layer has zero dependency on outer layers.

```
app/
├── domain/             Pure entities, DTOs, contracts (zero external deps)
│   ├── entities/       Provider, Policy, Log, Tester
│   ├── dto/            Cross-layer data transfer objects
│   ├── contracts/      Repository and adapter interfaces
│   └── utils/          Pure helpers (crypto, validation)
├── services/           Use Cases — business logic orchestration
│   ├── provider_service
│   ├── policy_service
│   ├── tester_service
│   └── stats_service
├── infrastructure/     I/O implementations
│   ├── database/       SQLAlchemy Async repositories + Alembic migrations
│   └── adapters/       Portkey HTTP adapter, Fernet crypto adapter
└── api/                FastAPI routes, schemas, middleware, DI
    ├── routes/         providers, policies, tester, logs, stats, settings, auth
    ├── schemas/        Pydantic v2 request / response models
    ├── middleware/     Auth, rate limiting, request logging
    └── dependencies/   FastAPI DI wiring
```

---

## Stack

| Layer | Technology |
|---|---|
| Backend | Python 3.12, FastAPI, Uvicorn |
| HTTP client | httpx (async) |
| Database | SQLite (WAL) for the POC, PostgreSQL-ready (asyncpg-compatible) |
| ORM | SQLAlchemy Async + aiosqlite, Alembic for migrations |
| Validation | Pydantic v2, pydantic-settings |
| Crypto | cryptography (Fernet symmetric encryption for provider keys) |
| Frontend | Next.js, TypeScript, Tailwind |
| Container | Docker multi-stage, docker-compose for local stack |
| Tooling | uv, pytest, mypy strict, ruff, bandit, pre-commit hooks |
| LLM gateway | Portkey (virtual keys, guardrail configs) |

---

## Highlights

```
~120  tests              unit + integration, contract-first
4     architecture       Clean Architecture, isolated layers
100%  production         AWS deployed, runbook handed over
0     plaintext secrets  Fernet encryption for all provider keys
```

---

## Quick start

Requires Python 3.12 and [uv](https://github.com/astral-sh/uv).

```bash
# 1. Configure environment
cp .env.example .env
# Edit .env: set PORTKEY_API_KEY, AUTH credentials, AUTH_SECRET, FERNET_KEY

# 2. Install dependencies
uv sync

# 3. Run migrations
uv run alembic upgrade head

# 4. Start the API
uv run uvicorn app.main:app --reload
# OpenAPI docs at http://localhost:8000/docs
```

### Full stack via Docker

```bash
docker compose up -d --build
# Backend on :8000, frontend on :3000
```

---

## Testing

```bash
uv run pytest -q                       # full suite
uv run pytest --cov=app                # with coverage
uv run mypy --strict app               # type check
uv run ruff check app                  # lint
uv run bandit -r app                   # security scan
```

Tests are contract-first: every route, repository and adapter has a colocated specification in the repo. Tests reference the contract, code is written against the same contract.

---

## Security

| Concern | Mitigation |
|---|---|
| Provider API key storage | Fernet symmetric encryption at rest; key derivation from a `FERNET_KEY` env variable |
| Authentication | JWT bearer tokens with configurable TTL, brute-force protection on the auth endpoint |
| Webhook trust | `X-Webhook-Secret` HMAC verification for inbound webhooks |
| Static analysis | `bandit` runs on every pre-commit |
| Dependency scanning | uv lockfile, pinned versions |

### Known limitation: rate limiter

The current brute-force rate limiter on the auth endpoint uses an in-memory `dict` per process. In multi-worker deployments (`uvicorn --workers > 1`, gunicorn) each worker keeps its own counter, so an attacker can distribute attempts across workers and effectively get `5 × N_workers` tries.

For production deployments with multiple workers, swap the limiter for a Redis-backed sliding window (`fastapi-limiter` or a Lua-based implementation on `redis-py`). For single-worker deployments the in-memory limiter is sufficient.

---

## Deploy

The repo ships with `deploy.sh` and `docker-entrypoint.sh` for AWS EC2 (or any Linux host with Docker). Production checklist:

1. Provision Ubuntu 24.04 with Docker installed.
2. Create `.env.production` with strong secrets (rotate `FERNET_KEY` once and store it in a secret manager).
3. Run `./deploy.sh` to build images and migrate the database.
4. Front the API with nginx or Caddy + Let's Encrypt for TLS.
5. Wire the audit log volume to durable storage (EBS or equivalent).
6. Schedule log rotation and database backups.

A complete runbook is delivered separately on engagement handover.

---

## Author

Built by Oleg Safronov. Senior Backend, AI Integration.
Portfolio: [github.com/safronovlab](https://github.com/safronovlab)

---

## License

Showcase repository. The codebase is published for portfolio review. No external license is currently issued.
