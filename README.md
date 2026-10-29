# OPNsense Fleet Manager Enterprise (OFME)

Centralized management for OPNsense firewalls (10 – 10 000+ devices): inventory,
monitoring, configuration, DHCP/DNS, policy-as-code, SD-WAN, zero-touch
provisioning, compliance, signed backups and an immutable audit trail.

| Layer | Technology |
|---|---|
| Backend | Python 3.13, FastAPI, SQLAlchemy 2.x (async), Alembic, Pydantic v2, Celery, Redis, gRPC |
| Frontend | React 18.3, TypeScript 5, Vite 6, MUI 6, TanStack Query 5, Redux Toolkit 2, React Router 6.28, Recharts 2 |
| Data | PostgreSQL 16, Redis 7, MinIO / S3 |
| Edge | `os-fleet-agent` OPNsense plugin (PHP 8.2, FreeBSD) |
| Delivery | Docker, Compose, Kubernetes, Helm, ArgoCD, GitHub Actions, GitLab CI |
| Observability | Prometheus, Grafana, Loki/Promtail, OpenTelemetry |

## Repository layout

```text
ofme/
├── backend/                  FastAPI application (DDD + Clean Architecture + CQRS)
│   ├── app/
│   │   ├── main.py           ASGI app factory, lifespan, routers, /metrics
│   │   ├── core/             config, security, logging, events, exceptions, pagination
│   │   ├── db/               Base, mixins, async/sync sessions
│   │   ├── domain/           entities, value objects, domain events, policies
│   │   ├── models/           SQLAlchemy ORM (40 tables)
│   │   ├── schemas/          Pydantic v2 DTOs
│   │   ├── repositories/     Repository pattern + Unit of Work
│   │   ├── services/         application services (use cases)
│   │   ├── cqrs/             command/query buses
│   │   ├── api/              deps, v1 routers/endpoints, WebSocket hub
│   │   ├── workers/          Celery app, beat schedule, task modules
│   │   ├── integrations/     OPNsense API, Git providers, S3, LDAP, OIDC, Vault
│   │   ├── grpc/             FleetRegistry service + generated stubs
│   │   ├── pki/              CA, issuance, rotation, CRL
│   │   ├── rbac/             permissions + role matrix
│   │   ├── observability/    metrics, OTel, middleware
│   │   └── seeds/            roles, permissions, bootstrap tenant/admin
│   ├── alembic/              migrations
│   ├── scripts/              export_openapi, generate_grpc, init_pki
│   └── tests/                pytest suite
├── frontend/                 React SPA (Vite, MUI, TanStack Query, Redux)
├── plugin/os-fleet-agent/    OPNsense PHP plugin (ports-style Makefile)
├── proto/fleet.proto         gRPC contract
├── api/openapi.{yaml,json}   exported OpenAPI 3.1
├── deploy/                   Kubernetes, Helm, ArgoCD (see deploy/README.md)
├── monitoring/               Prometheus rules, Grafana, Loki/Promtail, OTel
├── tests/load/               k6 load tests
├── docs/                     architecture, database, security, ops, …
├── .github/workflows/        GitHub Actions (CI, release)
├── .gitlab-ci.yml            GitLab pipeline
├── docker-compose.yml        full local stack
├── .env.example              environment template
└── Makefile                  developer entry points
```

## Quick start (Docker Compose)

```bash
cd ofme
cp .env.example .env
# edit .env: SECRET_KEY, POSTGRES_PASSWORD, MINIO_ROOT_PASSWORD,
#            S3_SECRET_KEY, BOOTSTRAP_ADMIN_PASSWORD
make up          # docker compose up -d --build
make ps
make logs-api
```

| URL | Service |
|---|---|
| `http://localhost:8080` | web console (nginx-served SPA, proxies `/api`, `/ws`) |
| `http://localhost:8000/docs` | OpenAPI UI (when `ENABLE_DOCS=true`) |
| `http://localhost:8000/api/v1/health/ready` | readiness (DB, Redis, S3, PKI) |
| `http://localhost:8000/metrics` | Prometheus metrics |
| `http://localhost:3000` | Grafana |
| `http://localhost:9090` | Prometheus |
| `http://localhost:9001` | MinIO console |

The API entrypoint waits for PostgreSQL and applies Alembic migrations
(`RUN_MIGRATIONS=true` by default); the lifespan then bootstraps the CA (if the
volume is empty), creates the S3 buckets, registers CQRS handlers, starts the
WebSocket hub, seeds the default tenant/admin when `BOOTSTRAP_ADMIN_PASSWORD`
is set, and re-aligns the built-in role matrix with `ROLE_MATRIX` so upgraded
deployments receive newly added permissions. `make migrate` is only needed if you
disable that.

First login: `BOOTSTRAP_ADMIN_EMAIL` / `BOOTSTRAP_ADMIN_PASSWORD`, then enrol TOTP
and change the password. Remove `BOOTSTRAP_ADMIN_PASSWORD` from `.env` afterwards.

The compose API runs **one uvicorn worker** because gRPC shares the lifespan;
scale with replicas, not workers. gRPC is disabled by default
(`GRPC_ENABLED=false`).

## Local development

```bash
make install        # backend venv + dev dependencies (uv)
make fe-install     # frontend dependencies
make test-db        # disposable PostgreSQL on :55432
make test           # pytest against PostgreSQL
make fe-dev         # Vite dev server on :5173
make lint typecheck fe-lint fe-test
```

Tests need PostgreSQL (models use `JSONB`/`PGUUID`); `TEST_DATABASE_URL` defaults
to the `make test-db` instance. See `backend/tests/conftest.py`.

---

## 🔒 Security Audit (2026-10-29)

Comprehensive security and architecture audit completed with **95 findings**:
- 🔴 10 Critical — Fix before any deployment
- 🟠 23 High — Fix before general availability
- 🟡 41 Medium — Fix within 30 days
- 🟢 21 Low — Plan for next quarter

### Full Report
📄 [`AUDIT_REPORT.md`](./AUDIT_REPORT.md) — Complete findings and code fixes  
📘 [`docs/SECURITY_README.md`](./docs/SECURITY_README.md) — Security quickstart & runbooks  
📋 [`docs/ACTION_PLAN.md`](./docs/ACTION_PLAN.md) — Prioritized remediation plan

### Quick Summary

| Category | Critical | High | Medium | Low |
|----------|----------|------|--------|-----|
| Backend Security | 4 | 5 | 8 | 6 |
| Frontend Security | 3 | 7 | 15 | 4 |
| Infrastructure | 3 | 4 | 6 | 3 |
| Architecture | 0 | 7 | 12 | 8 |
| **TOTAL** | **10** | **23** | **41** | **21** |

### P0: Critical (Do Today)
- [B-C1](./AUDIT_REPORT.md) TLS disabled in inventory service blocking/pause/unblock
- [B-C2](./AUDIT_REPORT.md) TLS disabled in device discovery
- [B-C3](./AUDIT_REPORT.md) Weak default SECRET_KEY in config
- [F-C1](./AUDIT_REPORT.md) JWT tokens stored in localStorage
- [I-C1](./AUDIT_REPORT.md) All Docker ports exposed to internet

### P0 Fixes
```bash
# 1. Generate new SECRET_KEY
openssl rand -hex 48 >> .env

# 2. Update frontend dependencies
cd frontend && npm install vitest@2.1.9 react-router-dom@7.18.5

# 3. Fix TLS in backend (edit these files)
# backend/app/services/inventory.py:190,229,267
# backend/app/services/devices.py:502

See [AUDIT_REPORT.md](./AUDIT_REPORT.md) for detailed instructions.
```

## Documentation

| Document | Path |
|---|---|
| Architecture, modules, C4, flows | [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) |
| Database schema, tables, indexes, retention | [`docs/DATABASE.md`](docs/DATABASE.md) |
| Security: trust boundaries, auth, RBAC, PKI | [`docs/SECURITY.md`](docs/SECURITY.md) |
| Deployment: compose, K8s, Helm, ArgoCD, DR | [`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md) |
| Operations: SLIs, alerts, runbooks, schedules | [`docs/OPERATIONS.md`](docs/OPERATIONS.md) |
| Policy-as-code reference | [`docs/POLICY_AS_CODE.md`](docs/POLICY_AS_CODE.md) |
| SD-WAN design and limits | [`docs/SD_WAN.md`](docs/SD_WAN.md) |
| Firmware status, upgrade and reboot | [`docs/FIRMWARE.md`](docs/FIRMWARE.md) |
| ZTP, enrollment and PKI binding | [`docs/ZTP.md`](docs/ZTP.md) |
| Specification-to-implementation traceability and gap register | [`docs/REQUIREMENTS_TRACEABILITY.md`](docs/REQUIREMENTS_TRACEABILITY.md) |
| Current state: what is deployed, verified and still limited | [`docs/STATE.md`](docs/STATE.md) |
| Deployment asset map | [`deploy/README.md`](deploy/README.md) |
| OpenAPI 3.1 | [`api/openapi.yaml`](api/openapi.yaml) |
| gRPC contract | [`proto/fleet.proto`](proto/fleet.proto) |
| OPNsense plugin | [`plugin/os-fleet-agent/README.md`](plugin/os-fleet-agent/README.md) |
| k6 load tests | [`tests/load/README.md`](tests/load/README.md) |

## Verification

```bash
make check      # ruff, pytest, eslint, vitest
make openapi    # regenerate api/openapi.{yaml,json} from the app
make grpc       # regenerate stubs from proto/fleet.proto
make helm-lint
make k8s-dry-run
```

`make test` expects the PostgreSQL container from `make test-db`; the suite skips
database-backed tests when it is unreachable.
