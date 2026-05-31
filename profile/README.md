<h1 align="center">Indu Sah Foundation</h1>

<p align="center">
  Engineering org for the foundation's website, API, and cloud infrastructure.
</p>

<p align="center">
  <a href="https://www.indusahfoundation.org"><b>Live site →</b></a>
</p>

---

## Repositories

| Repo | Stack | What it is |
|---|---|---|
| [**ISF-Frontend**](https://github.com/Indu-Sah-Foundation/ISF-Frontend) | React · TypeScript · Vite | Public website + admin dashboard. SPA hosted on Azure Static Web Apps. |
| [**ISF-Backend**](https://github.com/Indu-Sah-Foundation/ISF-Backend) | Go · Gin · PostgreSQL | API monolith (~20 domain packages) on Azure App Service B1. |
| [**ISF-Infastructure**](https://github.com/Indu-Sah-Foundation/ISF-Infastructure) | Terraform · Azure | All cloud resources as code, OIDC-authenticated GitHub Actions. |

## Stack

**Frontend** — React 19 · TypeScript · TanStack Router (file-based routing) · TanStack Query · TipTap (WYSIWYG editor) · Tailwind CSS v4 · Vite · Playwright (E2E) · Application Insights (web SDK)

**Backend** — Go 1.25 · Gin · PostgreSQL via pgx/v5 · Redis · golang-migrate (embedded migrations) · JWT (HS256) · bcrypt admin · Stripe Go SDK · Azure Translator REST · Azure Blob SDK · structured access logging

**Infrastructure** — Terraform · Azure App Service (B1) · Azure Static Web Apps · Azure Database for PostgreSQL Flexible Server · Azure Cache for Redis · Azure Blob Storage · Azure Container Registry · Azure Key Vault · Application Insights · Azure Translator

**CI/CD** — GitHub Actions · Azure OIDC (zero stored secrets) · Docker (distroless static binary) · plan-on-PR / apply-on-merge for Terraform · E2E tests gate every backend deploy · Playwright suite gates every frontend PR · branch protection (required checks block merge until green)

## Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│  Azure Static Web Apps   ←──  React SPA  ──→  Application Insights │
│                                  │              (page views, SPA    │
│                                  │               route tracking)    │
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
│    100+ langs)            sig + idempotency)         DB, Redis,    │
│                                                      API key)      │
└────────────────────────────────────────────────────────────────────┘
```

## Engineering highlights

- **Three-tier translation cache** — Redis → Postgres → Azure Translator with `<span class="notranslate">` placeholder protection. 80ms → 3ms warm reads across 100+ languages. Image-count + length-shrink sanity gates refuse to cache broken output. Detached write context so a reader bouncing mid-request never cancels the cache write.
- **Direct browser → Blob uploads** via short-lived SAS (write-only, 10-min TTL). Backend never touches image bytes — critical on B1.
- **Orphan-blob GC** sweeper diffs container against 7 reference tables and regex-scans markdown bodies. Dry-run vs apply via `GET` vs `POST`.
- **Stripe payment hardening** — webhook signature verification, idempotency via `donation_events(event_id)` table, friendly per-field validation errors. Live-mode keys sourced from Key Vault.
- **API-key gateway** — every request requires `X-API-Key`; rejects callers outside the frontend. `NoRoute` fallback runs the key check first, so unmatched paths return 401 to unauthenticated callers instead of leaking 404s. Exempt paths: Azure health probes, Stripe webhook.
- **JWT admin auth** — HS256 with explicit algorithm pinning (alg-confusion defense), 24h expiry, `RequireAuth` + `RequireRole` middleware composed for admin routes. `EnsureAdmin` re-hashes on boot so rotating the Key Vault password actually takes effect.
- **Per-IP token-bucket rate limits** on `/auth/login` (5/min), `/donations/checkout` (30/min), `/contacts` (5/min).
- **Embedded migrations** via `//go:embed`, run on boot through golang-migrate.
- **Distroless static container** — `CGO_ENABLED=0 -trimpath -ldflags="-s -w"`, runs as `nonroot`.
- **Liveness vs readiness split** — `/health` is always-200; `/ready` pings Postgres + Redis in parallel via goroutines.

### Frontend & quality

- **Playwright E2E suite** — ~316 tests across 5 engines (Chromium, Firefox, WebKit, Pixel, iPhone). Runs against the production static build with a fully mocked API layer (route-level interception, deterministic fixtures, no live backend). Gates every PR via branch protection.
- **Backend E2E tests** — in-process router against Postgres + Redis service containers in CI; gate every deploy.
- **Branch protection** — `main` blocks merge until required checks pass; strict (PR must be up-to-date) before merge; force-push and deletion disabled.
- **Production-only deploy** — PRs validate via build + E2E; only a push to `main` ships to the live site (no per-PR preview environments — avoids the Free-tier staging cap).
- **App-shell navbar** — transparent glass over the hero on home, solid white on inner pages, hide-on-scroll-down / reveal-on-scroll-up, responsive hamburger at `xl` so iPads get the mobile menu instead of an overflowing link row.
- **Responsive image pipeline** — WebP (desktop) / AVIF (mobile) hero variants with `<picture>` + `srcset` tiers (800/1400/2000/3000px), preloaded with `fetchpriority="high"` to cut first-paint time.
- **Frontend telemetry** — Application Insights web SDK with SPA route tracking, reporting to the same resource as the backend for correlated front-to-back diagnostics.
- **Article thumbnails** — admin-set poster (`<!-- thumbnail: url -->` marker) → first inline body image → local fallback, shared between the home blog cards and the stories listing.

## Repo layout (Backend)

```
internal/
├── achievements/   articles/      auth/         cache/        cleanup/
├── config/         contacts/      db/           e2e/          gallery/
├── health/         httperr/       middleware/   payments/     people/
├── projects/       storage/       team/         translate/    volunteers/
cmd/server/         cmd/migrate-images/  (one-shot Blogger image mirror)
                    cmd/hashpw/          (bcrypt hash CLI for admin pw rotation)
```

Every domain follows `repo → service → handler` with a `Repository` interface for testability.

## Authored by

[**Rohan Sah**](https://rohansah.dev) — founding software engineer. Wrote the frontend, backend, and infrastructure end-to-end.
