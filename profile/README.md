<h1 align="center">Indu Sah Foundation</h1>

<p align="center">
  Engineering org for the foundation's website, API, and cloud infrastructure.
</p>

<p align="center">
  <a href="https://orange-desert-0d758fb0f.7.azurestaticapps.net"><b>Live site →</b></a>
</p>

---

## Repositories

| Repo | Stack | What it is |
|---|---|---|
| [**ISF-Frontend**](https://github.com/Indu-Sah-Foundation/ISF-Frontend) | React · TypeScript · Vite | Public website + admin dashboard. SPA hosted on Azure Static Web Apps. |
| [**ISF-Backend**](https://github.com/Indu-Sah-Foundation/ISF-Backend) | Go · Gin · PostgreSQL | API monolith (~20 domain packages) on Azure App Service B1. |
| [**ISF-Infastructure**](https://github.com/Indu-Sah-Foundation/ISF-Infastructure) | Terraform · Azure | All cloud resources as code, OIDC-authenticated GitHub Actions. |

## Stack

**Frontend** — React 19 · TypeScript · TanStack Router (file-based routing) · TanStack Query · TipTap (WYSIWYG editor) · Tailwind CSS · Vite

**Backend** — Go 1.25 · Gin · PostgreSQL via pgx/v5 · Redis · golang-migrate (embedded migrations) · JWT (HS256) · bcrypt admin · Stripe Go SDK · Azure Translator REST · Azure Blob SDK · structured access logging

**Infrastructure** — Terraform · Azure App Service (B1) · Azure Static Web Apps · Azure Database for PostgreSQL Flexible Server · Azure Cache for Redis · Azure Blob Storage · Azure Container Registry · Azure Key Vault · Application Insights · Azure Translator

**CI/CD** — GitHub Actions · Azure OIDC (zero stored secrets) · Docker (distroless static binary) · plan-on-PR / apply-on-merge for Terraform · E2E tests gate every backend deploy

## Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│  Azure Static Web Apps   ←──  React SPA                            │
│                                  │                                 │
│                                  ▼  HTTPS + X-API-Key + JWT        │
│  Azure App Service (B1)  ←──  Go / Gin monolith                    │
│                                  │                                 │
│         ┌────────────────────────┼──────────────────────────┐      │
│         ▼                        ▼                          ▼      │
│   Postgres Flex          Cache for Redis            Blob Storage   │
│   (10 migrations,        (article + translation    (images,        │
│    pgxpool tuned          5-min TTL cache,          SAS uploads,   │
│    for B1)                rate-limit counters)      orphan GC)     │
│                                                                    │
│   Azure Translator       Stripe                     Key Vault      │
│   (HTML-aware pipeline,  (Checkout, webhook         (JWT, Stripe,  │
│    100+ langs)            sig + idempotency)         DB, Redis)    │
└────────────────────────────────────────────────────────────────────┘
```

## Engineering highlights

- **Three-tier translation cache** — Redis → Postgres → Azure Translator with `<span class="notranslate">` placeholder protection. 80ms → 3ms warm reads across 100+ languages. Image-count + length-shrink sanity gates refuse to cache broken output.
- **Direct browser → Blob uploads** via short-lived SAS (write-only, 10-min TTL). Backend never touches image bytes — critical on B1.
- **Orphan-blob GC** sweeper diffs container against 7 reference tables and regex-scans markdown bodies. Dry-run vs apply via `GET` vs `POST`.
- **Stripe payment hardening** — webhook signature verification, idempotency via `donation_events(event_id)` table, friendly per-field validation errors.
- **API-key gateway** — every request requires `X-API-Key`; rejects callers outside the frontend. Exempt paths: Azure health probes, Stripe webhook.
- **JWT admin auth** — HS256 with explicit algorithm pinning (alg-confusion defense), 24h expiry, `RequireAuth` + `RequireRole` middleware composed for admin routes.
- **Per-IP token-bucket rate limits** on `/auth/login` (5/min), `/donations/checkout` (30/min), `/contacts` (5/min).
- **Embedded migrations** via `//go:embed`, run on boot through golang-migrate.
- **E2E tests** — in-process router against Postgres + Redis service containers in CI, gate every deploy.
- **Distroless static container** — `CGO_ENABLED=0 -trimpath -ldflags="-s -w"`, runs as `nonroot`.
- **Liveness vs readiness split** — `/health` is always-200; `/ready` pings Postgres + Redis in parallel via goroutines.

## Repo layout (Backend)

```
internal/
├── achievements/   articles/      auth/         cache/        cleanup/
├── config/         contacts/      db/           e2e/          gallery/
├── health/         httperr/       middleware/   payments/     people/
├── projects/       storage/       team/         translate/    volunteers/
cmd/server/         cmd/migrate-images/  (one-shot Blogger image mirror)
```

Every domain follows `repo → service → handler` with a `Repository` interface for testability.

## Authored by

**Rohan Sah** — founding software engineer. Wrote the frontend, backend, and infrastructure end-to-end.
