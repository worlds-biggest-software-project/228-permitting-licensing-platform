# Permitting & Licensing Platform — Phased Development Plan

> Project: 228-permitting-licensing-platform · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises the candidate research (`research.md`, `features.md`, `standards.md`, `README.md`) and Data Model Suggestion 1 (Entity-Centric Normalised Relational) into an executable, phased delivery plan for an AI-native, open-source government permitting and licensing platform.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary backend language | Python 3.12 | The platform's defining differentiators are AI features (plan review, eligibility assistant, completeness checking) and geospatial work. Python has the strongest ecosystem for both: official OpenAI/Anthropic SDKs, LangChain, PyMuPDF/pdfplumber for plan parsing, ifcopenshell for IFC ingest, GeoPandas/Shapely/PyProj for parcel work, and rasterio for plan imagery. Government IT teams also accept Python more readily than Go/Rust. |
| API framework | FastAPI 0.115+ | Native async (essential for AI/webhook-heavy workloads), auto-generated OpenAPI 3.1 spec (a standard called out in `standards.md`), first-class Pydantic v2 validation matching JSON Schema Draft 2020-12, and excellent performance via Starlette/uvicorn. Aligns with NEPA permitting data standard's recommended JSON/YAML/OpenAPI format. |
| Frontend framework | Next.js 15 (App Router) + React 18 + TypeScript | Provides server components for accessibility-critical pages, hybrid SSR/CSR for the constituent portal, strong WCAG 2.1 AA tooling (axe-core, eslint-plugin-jsx-a11y), and a mature ecosystem for maps (MapLibre GL JS), forms (react-hook-form + zod), and tables (TanStack Table). Three distinct UI surfaces (constituent portal, staff console, inspector mobile PWA) share a single Next.js codebase. |
| Component / design system | shadcn/ui + Tailwind CSS v4 + Radix primitives | Radix gives WCAG-compliant primitives out of the box (the most important constraint for an ADA-Title-II-targeted product); shadcn/ui supplies copy-in-code components we can audit and customise; Tailwind keeps styling consistent across three apps without runtime CSS-in-JS overhead. |
| Mobile inspector app | PWA (offline-first) — Next.js + service worker + IndexedDB via Dexie.js | A PWA avoids App Store gating, supports offline inspection capture (a Cloudpermit differentiator we must match), and reuses the web codebase. Native wrappers (Capacitor) can be added later without rewriting business logic. |
| Primary database | PostgreSQL 16 + PostGIS 3.4 | Data Model Suggestion 1 is fully relational (38+ tables, foreign keys, CHECK constraints) and requires spatial queries on parcels. PostgreSQL also gives us JSONB (used for `form_data` and `validation_rules`), partial indexes, partitioning (for `audit_log`), and Row Level Security (RLS) to enforce `jurisdiction_id` tenant isolation. |
| ORM / query layer | SQLAlchemy 2.0 (async) + Alembic | Mature, async-capable, well-supported PostGIS extensions via GeoAlchemy2. Alembic provides reviewable, reversible schema migrations — critical because government IT will demand audit trails on schema changes. |
| Schema validation | Pydantic v2 | Powers FastAPI request/response models, generates JSON Schema Draft 2020-12 (a standard explicitly referenced), and shares schemas with the OpenAPI generator. |
| Async task queue | Celery 5 with Redis broker and result backend | Long-running AI calls (plan review, completeness checks), webhook delivery, notification fan-out, and BLDS export generation all need to run outside the request cycle. Celery is the most production-hardened option for Python, with built-in retries, exponential backoff, and scheduled tasks (celery beat for licence expiry scans). |
| Cache + queue broker | Redis 7 | Used for Celery broker, session cache, rate limiting, and short-lived idempotency keys for payment webhooks. |
| Object storage | S3-compatible (MinIO for self-hosted, AWS S3 / R2 / Azure Blob via abstraction) | Plan documents, inspection photos, IFC models, and exported BLDS bundles need durable blob storage. Abstracting behind a `BlobStore` interface lets jurisdictions choose their provider while supporting self-hosted MinIO out of the box for the open-source default. |
| Authentication / authorisation | Authlib (OIDC client) + custom OAuth 2.0 server (Authlib server components) | Must support Login.gov (NIST SP 800-63-3 IAL2 OIDC) for federal alignment, agency SSO (Azure AD / Okta via generic OIDC), and a built-in local IdP for jurisdictions without one. Custom OAuth 2.0 server exposes API tokens for third-party developers (mirroring Accela Construct API). |
| RBAC enforcement | Custom permission middleware backed by `role_permissions` table + PostgreSQL RLS | Two layers of defence: application-level checks before each handler, and database-level RLS policies that enforce `jurisdiction_id` scoping even if application logic is bypassed. |
| AI / LLM integration | Anthropic Claude (Opus 4 + Haiku 4) via official SDK; pluggable provider via abstraction | Claude leads on long-context document analysis (plans can be 50+ pages of mixed text and diagrams). A `LLMProvider` interface lets agencies swap to Azure OpenAI, Vertex AI, or self-hosted (Ollama / vLLM) for data-residency-strict deployments. |
| LLM orchestration | Direct SDK + Pydantic-AI for structured output | LangChain adds unnecessary indirection for our limited use cases (completeness check, eligibility assistant, plan review extraction). Pydantic-AI gives us typed, validated LLM responses without the LangChain surface area. |
| Vector search | pgvector (PostgreSQL extension) | We need embedding search on permit history (for the eligibility assistant) and on code rule corpora (for plan review). Keeping vectors in PostgreSQL avoids a second datastore for the MVP. |
| PDF / plan parsing | PyMuPDF (fitz) + pdfplumber + Pillow | PyMuPDF for fast text extraction and rendering; pdfplumber for table extraction; Pillow for image conversion before vision-LLM analysis. |
| IFC / BIM ingest | ifcopenshell 0.8 | The dominant open-source library for IFC parsing; required for the v1.2 BIM/IFC permit pathway. |
| Geospatial libraries | GeoAlchemy2, Shapely, PyProj | Standard Python geospatial stack; integrates with PostGIS column types. |
| Map rendering (frontend) | MapLibre GL JS + Protomaps (PMTiles) | Open-source vector tile rendering, no Mapbox licence fees. PMTiles allows self-hosting tile data. Esri ArcGIS is also supported via a pluggable basemap adapter. |
| Plan review markup viewer | PSPDFKit alternative — pdf.js + custom annotation layer | pdf.js (Mozilla) renders plans in browser; we layer a custom annotation canvas storing markups in our `plan_review_comments` table. BCF export for round-tripping with Bluebeam. |
| Payment processing | Stripe (default), with pluggable processors (Authorize.Net, PayPal, GovTech Pay) via `PaymentProcessor` interface | Stripe has the cleanest API and webhook story for an open-source default. Plug-in architecture lets jurisdictions use locked-in govtech processors. PCI scope stays minimal — no card data ever touches our servers. |
| Email | SMTP via aiosmtplib (default) + provider adapters (SendGrid, AWS SES, Postmark) | Most government deployments mandate a specific outbound provider; abstract the interface. |
| SMS | Twilio (default) + pluggable provider (AWS SNS, TextMyGov, Bandwidth) | Same rationale as email. |
| Search | PostgreSQL full-text search (tsvector) for MVP; OpenSearch as v1.2 upgrade | tsvector is sufficient for permit number / address / applicant name search. OpenSearch becomes worthwhile only at multi-jurisdiction scale or for analytics queries. |
| Observability | OpenTelemetry SDK → Grafana stack (Loki logs, Tempo traces, Prometheus metrics, Mimir long-term metrics) | All open-source, self-hostable, integrates with managed services (Grafana Cloud, Honeycomb, Datadog) via OTLP. |
| Containerisation | Docker + docker-compose (dev/single-tenant) + Helm chart (Kubernetes for multi-tenant SaaS) | Self-hosted deployments use docker-compose; cloud SaaS uses the Helm chart. Both reference the same images. |
| CI/CD | GitHub Actions | Free for public repos, ubiquitous, supports our Docker / pytest / npm test pipeline. |
| Backend testing | pytest, pytest-asyncio, pytest-cov, hypothesis (property tests for fee calculation), respx (HTTP mocking), testcontainers (PostgreSQL/Redis in CI) | Standard async Python testing stack; testcontainers ensures CI runs against real PostgreSQL/PostGIS rather than SQLite-substituted tests. |
| Frontend testing | Vitest (unit), Playwright (E2E + accessibility), Storybook + Chromatic (component visual review), axe-core via @axe-core/playwright (automated WCAG 2.1 AA checks) | Playwright runs accessibility audits in CI so we never regress on ADA Title II conformance. |
| API contract testing | Schemathesis | Property-based test generator from OpenAPI spec; ensures the API actually conforms to the published spec — critical for a public-facing government API. |
| Linting / formatting | ruff (Python lint+format), mypy (type check), eslint + prettier (TS), biome as a fallback | ruff replaces black+isort+flake8 in a single tool; mypy in strict mode catches contract drift. |
| Package management | uv (Python) + pnpm (Node) | uv is dramatically faster than pip/poetry; pnpm reduces install times and disk usage for the monorepo's three frontend apps. |
| Monorepo tooling | Turborepo (frontend apps) + uv workspace (backend services) | Three Next.js apps share components; Turborepo gives us cached, parallel builds. The Python backend uses uv workspaces for shared core packages. |
| Documentation | MkDocs Material (architecture/operator docs) + auto-published OpenAPI Swagger UI + Storybook (frontend components) | Three distinct doc surfaces for three audiences: operators (MkDocs), API consumers (Swagger/Redoc), frontend contributors (Storybook). |
| Licence | Apache 2.0 | Permissive enough that government IT teams will adopt without legal review; provides explicit patent grant (more important than MIT for a domain with code-compliance algorithms). |

---

### Project Structure

```
permitting-platform/
├── README.md
├── LICENSE                                  (Apache 2.0)
├── CONTRIBUTING.md
├── SECURITY.md
├── .github/
│   ├── workflows/
│   │   ├── backend-ci.yml
│   │   ├── frontend-ci.yml
│   │   ├── e2e.yml
│   │   ├── accessibility.yml
│   │   ├── docker-publish.yml
│   │   └── release.yml
│   └── ISSUE_TEMPLATE/
├── docker-compose.yml                       (dev stack: postgres+postgis, redis, minio, mailhog, the app)
├── docker-compose.prod.yml
├── deploy/
│   ├── helm/permitting-platform/            (Kubernetes Helm chart)
│   └── terraform/                           (reference IaC modules)
├── docs/
│   ├── mkdocs.yml
│   └── src/
│       ├── architecture/
│       ├── operators/
│       ├── api/                             (curated guides; spec auto-generated)
│       ├── data-model/
│       └── ai-features/
├── backend/
│   ├── pyproject.toml                       (uv workspace root)
│   ├── uv.lock
│   ├── Dockerfile
│   ├── .env.example
│   ├── alembic.ini
│   ├── migrations/                          (Alembic migrations)
│   │   └── versions/
│   ├── seed/                                (seed jurisdictions, permit types, BLDS templates)
│   ├── src/
│   │   └── permitting/
│   │       ├── __init__.py
│   │       ├── main.py                      (FastAPI app factory)
│   │       ├── config.py                    (Pydantic Settings)
│   │       ├── deps.py                      (FastAPI dependencies)
│   │       ├── core/                        (cross-cutting: logging, errors, ids, time, money)
│   │       ├── db/                          (engine, session, base models, RLS helpers)
│   │       │   ├── models/                  (SQLAlchemy ORM models — one file per domain area)
│   │       │   └── repositories/            (data access layer)
│   │       ├── schemas/                     (Pydantic request/response schemas)
│   │       ├── api/
│   │       │   ├── v1/
│   │       │   │   ├── auth.py
│   │       │   │   ├── jurisdictions.py
│   │       │   │   ├── parcels.py
│   │       │   │   ├── permits.py
│   │       │   │   ├── permit_types.py
│   │       │   │   ├── workflows.py
│   │       │   │   ├── inspections.py
│   │       │   │   ├── fees.py
│   │       │   │   ├── payments.py
│   │       │   │   ├── documents.py
│   │       │   │   ├── plan_review.py
│   │       │   │   ├── licences.py
│   │       │   │   ├── code_enforcement.py
│   │       │   │   ├── notifications.py
│   │       │   │   ├── ai_assistant.py
│   │       │   │   ├── analytics.py
│   │       │   │   └── public/              (unauthenticated endpoints: BLDS export, Open311)
│   │       │   └── webhooks.py
│   │       ├── services/                    (business logic — one module per bounded context)
│   │       │   ├── permits/
│   │       │   ├── workflows/
│   │       │   ├── inspections/
│   │       │   ├── fees/
│   │       │   ├── plan_review/
│   │       │   ├── code_enforcement/
│   │       │   └── licensing/
│   │       ├── integrations/                (external systems)
│   │       │   ├── payments/                (stripe.py, authorize_net.py, base.py)
│   │       │   ├── email/
│   │       │   ├── sms/
│   │       │   ├── blob_store/              (s3.py, minio.py, local.py, base.py)
│   │       │   ├── gis/                     (esri.py, openlayers.py, base.py)
│   │       │   ├── login_gov.py
│   │       │   └── open311.py
│   │       ├── ai/
│   │       │   ├── providers/               (anthropic.py, openai.py, local.py, base.py)
│   │       │   ├── prompts/                 (versioned prompt templates)
│   │       │   ├── completeness.py
│   │       │   ├── eligibility.py
│   │       │   ├── plan_review.py
│   │       │   ├── scheduling.py
│   │       │   └── embeddings.py
│   │       ├── geospatial/                  (PostGIS helpers, parcel lookup)
│   │       ├── exports/
│   │       │   ├── blds.py                  (BLDS CSV export)
│   │       │   ├── open311.py
│   │       │   └── analytics.py
│   │       ├── tasks/                       (Celery task definitions)
│   │       │   ├── celery_app.py
│   │       │   ├── ai_tasks.py
│   │       │   ├── notification_tasks.py
│   │       │   ├── export_tasks.py
│   │       │   └── scheduled.py             (beat schedules: licence expiry, SLA escalations)
│   │       ├── auth/
│   │       │   ├── oidc.py
│   │       │   ├── oauth_server.py
│   │       │   ├── rbac.py
│   │       │   └── tenancy.py               (jurisdiction context resolution)
│   │       ├── audit/                       (audit_log writer, decorator, query)
│   │       └── observability/               (otel setup, logging config)
│   └── tests/
│       ├── conftest.py                      (testcontainers fixtures, factory_boy)
│       ├── unit/
│       ├── integration/
│       ├── api/
│       ├── e2e/
│       └── fixtures/                        (sample permits, BLDS golden files, sample IFC models)
├── frontend/
│   ├── package.json                         (pnpm workspace root)
│   ├── pnpm-workspace.yaml
│   ├── turbo.json
│   ├── tsconfig.base.json
│   ├── apps/
│   │   ├── portal/                          (constituent/applicant portal)
│   │   │   ├── package.json
│   │   │   ├── next.config.ts
│   │   │   └── src/
│   │   │       ├── app/                     (App Router)
│   │   │       ├── components/
│   │   │       ├── lib/                     (api client, hooks)
│   │   │       └── messages/                (i18n catalogs)
│   │   ├── staff/                           (agency staff console)
│   │   │   └── src/...
│   │   └── inspector/                       (inspector PWA — offline-first)
│   │       └── src/...
│   ├── packages/
│   │   ├── ui/                              (shadcn/ui + custom components)
│   │   ├── api-client/                      (auto-generated from OpenAPI spec)
│   │   ├── forms/                           (dynamic form renderer for permit_type_fields)
│   │   ├── maps/                            (MapLibre wrappers, parcel picker)
│   │   ├── plan-viewer/                     (pdf.js + annotation canvas)
│   │   ├── i18n/
│   │   └── tsconfig/
│   └── tests/
│       ├── e2e/                             (Playwright)
│       └── a11y/                            (axe-core via Playwright)
├── tools/
│   ├── openapi-export.py                    (dumps spec for client generation)
│   ├── seed_dev_data.py
│   └── load-test/                           (k6 scripts)
└── examples/
    ├── sample-permit-types/                 (JSON: building, electrical, plumbing, business licence)
    ├── sample-fee-schedules/
    ├── sample-workflow-templates/
    └── blds-export-golden/                  (reference BLDS CSVs for tests)
```

---

## Phase 1: Foundation, Tenancy, and Identity

### Purpose
Stand up the repository, container stack, database schema baseline (jurisdictions, departments, users, RBAC), and authentication primitives. After Phase 1, a developer can run `docker-compose up`, log in as a seeded admin in a seeded jurisdiction, and call an authenticated `/whoami` endpoint that resolves the jurisdiction context and role correctly. Nothing in later phases is allowed to bypass the tenancy and audit primitives established here.

### Tasks

#### 1.1 — Monorepo, tooling, and CI scaffolding

**What**: Initialise the repository with backend (uv + FastAPI) and frontend (pnpm + Turborepo) workspaces, ruff/mypy/eslint/prettier configurations, a docker-compose dev stack (Postgres+PostGIS, Redis, MinIO, MailHog), and GitHub Actions workflows.

**Design**:
- `backend/pyproject.toml` declares the workspace and pins runtime to Python 3.12; lockfile is `uv.lock`.
- `frontend/pnpm-workspace.yaml` declares `apps/*` and `packages/*`.
- `docker-compose.yml` services:
  - `postgres` — image `postgis/postgis:16-3.4`, volume-mounted, healthcheck `pg_isready`.
  - `redis` — image `redis:7-alpine`.
  - `minio` — image `minio/minio:latest` with default bucket `permitting-dev` created via init script.
  - `mailhog` — image `mailhog/mailhog` (SMTP capture for local).
  - `api` — built from `backend/Dockerfile`, depends on postgres+redis+minio.
  - `worker` — same image, command `celery -A permitting.tasks worker`.
  - `scheduler` — `celery -A permitting.tasks beat`.
- `.env.example` lists all required variables with safe dev defaults. `Settings` (Pydantic) reads from env with explicit types.
- CI workflows:
  - `backend-ci.yml` — `uv sync`, `ruff check`, `ruff format --check`, `mypy --strict`, `pytest` with testcontainers.
  - `frontend-ci.yml` — `pnpm install --frozen-lockfile`, `pnpm turbo run lint test build`.
  - `docker-publish.yml` — builds and pushes multi-arch images on tag pushes.

**Testing**:
- `Unit: Settings loads from env with all defaults → no validation errors`
- `Unit: Settings with missing required SECRET_KEY → ValidationError mentions SECRET_KEY`
- `Integration: docker-compose up brings postgres/redis/minio to healthy state in <60s` (CI smoke test using `compose up --wait`)
- `Integration: /healthz returns {"status":"ok"} and includes db/redis/blob_store checks`
- `CI: ruff/mypy/pytest must pass on a freshly cloned repo with only uv and docker installed`

#### 1.2 — Database baseline, Alembic, and RLS policy framework

**What**: Create initial Alembic migration covering `jurisdictions`, `departments`, `users`, `user_department_roles`, `role_permissions`, `audit_log`. Establish multi-tenancy via `jurisdiction_id` foreign keys and PostgreSQL Row Level Security policies.

**Design**:
- Migration `0001_baseline.sql` adapts the DDL from Data Model Suggestion 1 sections "Core Identity & Multi-Tenancy", "User & Access Management", and "Audit Log".
- Each tenant-scoped table receives an RLS policy:
  ```sql
  ALTER TABLE departments ENABLE ROW LEVEL SECURITY;
  CREATE POLICY tenant_isolation ON departments
      USING (jurisdiction_id = current_setting('app.current_jurisdiction')::uuid);
  ```
- A `db.session.set_tenant_context(jurisdiction_id, user_id)` helper executes `SELECT set_config('app.current_jurisdiction', $1, true)` and the analogous `app.current_user` setting at the start of each request transaction.
- Two PostgreSQL roles created in seed: `permitting_app` (used by API) and `permitting_admin` (used by migrations and BLDS export jobs, bypasses RLS).
- `Audit` SQLAlchemy event listeners capture `before_insert`/`before_update`/`before_delete` for all models registered in `audit_registry`, writing to `audit_log` with the diff in `old_values`/`new_values`.

**Testing**:
- `Unit: set_tenant_context with valid UUID → subsequent SELECT scoped to jurisdiction (verified by inserting cross-tenant rows and asserting they're invisible)`
- `Unit: SELECT without set_tenant_context as permitting_app role → zero rows returned`
- `Integration: alembic upgrade head then alembic downgrade base → schema returns to empty`
- `Integration: insert + update + delete on a department → 3 corresponding audit_log rows with correct old_values/new_values JSON`
- `Integration: as permitting_app, attempt SELECT on departments of a different jurisdiction_id → 0 rows (RLS enforcement)`
- `Property test (hypothesis): random JSON diffs round-trip through audit_log without data loss`

#### 1.3 — OIDC authentication and local IdP

**What**: Implement OIDC client supporting Login.gov and generic OIDC providers (Azure AD, Okta), and a built-in local username/password IdP. Establish session management via signed JWT access tokens + opaque refresh tokens stored in Redis.

**Design**:
- `auth/oidc.py` exposes:
  ```python
  class OIDCProvider:
      name: str
      issuer: str
      client_id: str
      client_secret: SecretStr
      scopes: list[str]
      acr_values: str | None  # e.g., "http://idmanagement.gov/ns/assurance/ial/2"
  ```
- Endpoints:
  - `GET /api/v1/auth/providers` → list configured providers.
  - `GET /api/v1/auth/{provider}/login` → 302 to authorization endpoint with PKCE.
  - `GET /api/v1/auth/{provider}/callback` → exchanges code, upserts `users` row (matching on `identity_provider` + `identity_provider_id`), issues tokens, sets HttpOnly+Secure+SameSite=Lax cookie.
  - `POST /api/v1/auth/login` (local) → email+password, bcrypt verify, issues tokens.
  - `POST /api/v1/auth/refresh` → rotates refresh token.
  - `POST /api/v1/auth/logout` → revokes refresh, clears cookie.
  - `GET /api/v1/me` → returns user + active jurisdictions + roles.
- Access token JWT claims: `sub` (user_id), `jur` (active jurisdiction_id), `roles` (list of `dept_code:role`), `iat`, `exp`, `aud`.
- Login.gov configuration requires PKCE (S256), JWT client assertion (RS256), and `acr_values` set to `http://idmanagement.gov/ns/assurance/ial/2`.
- Refresh tokens are opaque 256-bit secrets, stored in Redis as `refresh:{sha256}` → `{user_id, jurisdiction_id, expires_at}` with 30-day TTL.

**Testing**:
- `Unit: build_authorize_url(provider) → URL contains required PKCE params and state`
- `Unit: verify_access_token(valid) → claims; verify_access_token(expired) → AuthError`
- `Integration (mocked OIDC): callback with valid code → user upserted, tokens issued, cookie set`
- `Integration (mocked OIDC): callback with mismatched state → 400`
- `Integration: local login with wrong password → 401 and audit_log entry with action='login_failed'`
- `Integration: refresh with stolen-then-rotated token → 401 (rotation detection)`
- `E2E (Playwright): user clicks Login.gov button → mock provider → returns to portal authenticated`

#### 1.4 — RBAC engine and `Requires` dependency

**What**: Implement role-permission evaluation backed by `role_permissions` and a reusable FastAPI dependency for endpoint-level enforcement.

**Design**:
- `auth/rbac.py`:
  ```python
  @dataclass(frozen=True)
  class Permission:
      resource: str        # "permits", "inspections", "fees", ...
      action: str          # "read", "create", "update", "delete", "approve"

  class Requires:
      def __init__(self, *permissions: Permission, any: bool = False): ...
      def __call__(self, principal: Principal = Depends(get_principal)) -> Principal: ...
  ```
- Usage in routes: `def list_permits(p: Principal = Depends(Requires(Permission("permits","read"))))`.
- `Principal` carries `user_id`, `jurisdiction_id`, set of `(resource, action)` resolved at login time and cached in Redis with 5-minute TTL keyed by user+jurisdiction. Cache is invalidated when `user_department_roles` or `role_permissions` mutate (SQLAlchemy event).
- Seed `role_permissions` with a default matrix for the six roles (`viewer`, `clerk`, `reviewer`, `inspector`, `supervisor`, `admin`) covering all resources defined in later phases.

**Testing**:
- `Unit: Requires(Permission("permits","read")) on principal lacking that permission → HTTPException(403)`
- `Unit: Requires(p1, p2) (all-of) and Requires(p1, p2, any=True) (any-of) behave correctly`
- `Integration: change user role → cache invalidated → next request reflects new permissions`
- `Integration: as inspector, GET /api/v1/permits → 403; as clerk, GET /api/v1/permits → 200`

#### 1.5 — Frontend shell and authenticated layout

**What**: Bootstrap the three Next.js apps (`portal`, `staff`, `inspector`) sharing `packages/ui`, `packages/api-client`, `packages/i18n`. Implement login flow, authenticated layout shell, and jurisdiction switcher.

**Design**:
- `packages/api-client` is generated via `openapi-typescript-codegen` from the FastAPI-exported `openapi.json`; build script: `pnpm run generate:client`.
- Each app has a `middleware.ts` that redirects unauthenticated requests to `/login` and refreshes access tokens transparently using the refresh cookie.
- Shared `<AuthProvider>` exposes `useAuth()` → `{ user, jurisdiction, roles, login, logout, switchJurisdiction }`.
- `<JurisdictionSwitcher>` is rendered in the staff app header when user has roles across multiple jurisdictions.
- Tailwind theme tokens in `packages/ui/src/theme.ts` drive light/dark mode; `prefers-reduced-motion` respected.

**Testing**:
- `E2E (Playwright): unauthenticated visit to /staff/dashboard → redirect to /login`
- `E2E (Playwright): successful login → land on /staff/dashboard with user name visible in header`
- `E2E (Playwright): switching jurisdiction reissues access token with new jur claim and reloads scoped data`
- `A11y (axe-core): /login passes WCAG 2.1 AA with zero violations`
- `A11y (axe-core): authenticated /staff layout passes WCAG 2.1 AA with zero violations`

---

## Phase 2: Configuration Domain — Parcels, Permit Types, Fee Schedules, Workflows

### Purpose
Build the configuration substrate that every operational workflow depends on. After Phase 2, an admin can define parcels (via import or manual entry), permit types with custom form fields and required documents, fee schedules and items, and workflow templates with steps and dependencies. Nothing yet operates on a real permit, but the platform is now configurable for any jurisdiction.

### Tasks

#### 2.1 — Parcel model, import, and geospatial lookup

**What**: Implement `parcels` and `parcel_owners` tables (DDL from data-model-suggestion-1), bulk import (CSV/GeoJSON/Shapefile), and lookup endpoints by APN, address, and lat/lng radius.

**Design**:
- SQLAlchemy models in `db/models/parcels.py` with GeoAlchemy2 `Geometry('POLYGON', srid=4326)` column.
- Import pipeline at `services/parcels/import.py`:
  ```python
  class ParcelImportSource(Enum):
      CSV = "csv"
      GEOJSON = "geojson"
      SHAPEFILE = "shapefile"
      ESRI_FEATURE_SERVICE = "esri_feature_service"

  def import_parcels(source: ParcelImportSource, payload: bytes | str, field_mapping: dict) -> ImportResult
  ```
- Endpoints:
  - `POST /api/v1/parcels/import` (multipart, async Celery task, returns job id).
  - `GET /api/v1/parcels?search=<text>&page=...` — full-text on address + parcel_number.
  - `GET /api/v1/parcels/{id}` — full detail + owners + active permits count.
  - `GET /api/v1/parcels/lookup?lat=&lng=&radius_m=` — PostGIS `ST_DWithin`.
  - `GET /api/v1/parcels/{id}/zoning` — returns zoning_code + description.
- Import job writes to `import_jobs` table with status/progress/errors_csv URL.

**Testing**:
- `Unit: CSV import with malformed row → error captured in errors_csv, valid rows still inserted`
- `Unit: GeoJSON import preserves polygon SRID 4326`
- `Integration (PostGIS): lookup at known coordinates returns expected parcel`
- `Integration: import 10k parcels → completes in <60s, all retrievable by APN`
- `API: GET /api/v1/parcels/lookup with invalid coordinates → 400`
- `E2E: admin uploads sample-parcels.csv → import job completes → search by address returns row`

#### 2.2 — Permit type configuration

**What**: Implement `permit_types`, `permit_type_fields`, `permit_type_required_documents` with full CRUD plus a JSON-schema export of the configured form for the frontend renderer.

**Design**:
- Pydantic schemas in `schemas/permit_types.py` mirror DDL with discriminated unions on `field_type`:
  ```python
  class TextFieldDef(BaseModel):
      field_type: Literal["text"]
      validation: TextValidation  # {min_length, max_length, pattern}

  class SelectFieldDef(BaseModel):
      field_type: Literal["select"]
      options: list[FieldOption]

  PermitTypeFieldDef = Annotated[Union[TextFieldDef, SelectFieldDef, ...], Field(discriminator="field_type")]
  ```
- Endpoints under `/api/v1/permit-types/`:
  - `GET /` list, `POST /` create, `GET /{id}` detail, `PUT /{id}` update, `DELETE /{id}` soft-delete.
  - `GET /{id}/form-schema` returns a JSON Schema Draft 2020-12 document the frontend renders dynamically.
- Seed examples in `examples/sample-permit-types/`: `building_new.json`, `electrical.json`, `plumbing.json`, `business_licence.json`.

**Testing**:
- `Unit: validation_rules.min > max → ValidationError`
- `Unit: form_schema generator produces valid JSON Schema (validated via jsonschema package)`
- `Integration: create permit type with 20 fields → all retrievable in display_order`
- `API: PUT to update field labels → audit_log records both old and new values`
- `E2E: admin builds a "New Residential" permit type via drag-drop UI → saved → fetched → renders identical form`

#### 2.3 — Fee schedule and fee calculation engine

**What**: Implement `fee_schedules`, `fee_items`, and a deterministic fee calculation service supporting flat, per-sqft, per-unit, percentage, tiered, and formula calculations.

**Design**:
- `services/fees/calculator.py`:
  ```python
  @dataclass
  class FeeCalculationContext:
      permit_type_id: UUID
      jurisdiction_id: UUID
      form_data: dict
      estimated_cost: Decimal | None
      square_footage: Decimal | None
      unit_count: int | None

  def calculate_fees(ctx: FeeCalculationContext, fee_schedule_id: UUID | None = None) -> list[CalculatedFee]
  ```
- Formula evaluation uses a sandboxed expression evaluator (no `eval`): a small parser supporting `+ - * / ()`, comparison ops, `CASE WHEN`, and references to context fields. Implementation: `services/fees/formula.py` with a pratt parser; reject any identifier not in an allowlist.
- Endpoint: `POST /api/v1/permit-types/{id}/preview-fees` accepts `form_data` and returns calculated fees without persisting.
- Fee schedules are versioned by `effective_date`; calculation uses the schedule active on the permit's `applied_date`.

**Testing**:
- `Unit: flat fee → exact amount returned`
- `Unit: per_sqft × 2500 sqft × $1.25 → $3,125`
- `Unit: percentage of valuation with min_amount enforced (low valuation) → min applied`
- `Unit: tiered fee with brackets [0-50k:flat 100, 50k+:0.2%] → correct tier selected`
- `Unit: formula 'CASE WHEN valuation < 50000 THEN 100 ELSE valuation * 0.002 END' with valuation=75000 → 150.00`
- `Unit: formula referencing undefined identifier → FormulaError before evaluation`
- `Property test (hypothesis): random valuations across all calculation types → result is always Decimal, never negative, respects min/max bounds`
- `Integration: preview-fees endpoint returns identical result for same input across 100 invocations (determinism)`

#### 2.4 — Workflow template designer and engine bootstrap

**What**: Implement `workflow_templates`, `workflow_template_steps`, `workflow_step_dependencies`, and the workflow instantiation service. The runtime engine that advances workflows lives in Phase 3.

**Design**:
- Pydantic schemas validate that step graph is a DAG (cycle detection via Kahn's algorithm at save time).
- Endpoints under `/api/v1/workflows/templates/`: CRUD plus `POST /{id}/validate` returning structural issues.
- `services/workflows/instantiate.py`:
  ```python
  def instantiate_workflow(permit_id: UUID, template_id: UUID) -> WorkflowInstance
  ```
  creates a `workflow_instances` row plus a `workflow_step_instances` row per template step in `pending` status, computing `due_date` from `sla_days`.
- Seed templates in `examples/sample-workflow-templates/`: `residential_building.json` (intake → completeness check → zoning review → building review → fire review [parallel] → fee assessment → issue), `business_licence.json` (intake → review → background check → issue).

**Testing**:
- `Unit: template with cycle (A→B→A) → ValidationError("workflow contains cycle")`
- `Unit: template with orphaned step (no path from intake) → ValidationError`
- `Unit: instantiate_workflow on permit → expected step_instances created, first step in 'in_progress' if no dependencies`
- `Integration: PUT to template adds a step → existing instances unaffected, new permits get new template`

---

## Phase 3: Permit Application Lifecycle — Intake, Workflow Execution, Status

### Purpose
Phase 3 delivers the core value: an applicant can submit a permit application through the portal; the workflow engine routes it through configured steps; staff reviewers can act on assigned steps; status transitions are tracked, audited, and reflected in real time. After Phase 3 the platform processes permits end-to-end without payment, inspections, or AI — those layer on in Phase 4+.

### Tasks

#### 3.1 — Permit CRUD and form data validation

**What**: Implement `permits`, `permit_status_history`, `permit_contractors`, `contractors`, with endpoints for draft creation, update, submission, and retrieval.

**Design**:
- Status state machine in `services/permits/status.py`:
  ```python
  ALLOWED_TRANSITIONS: dict[str, set[str]] = {
      "draft": {"submitted", "withdrawn"},
      "submitted": {"under_review", "withdrawn", "denied"},
      "under_review": {"revisions_requested", "approved", "denied"},
      "revisions_requested": {"under_review", "withdrawn"},
      "approved": {"issued"},
      "issued": {"inspection_scheduled", "completed", "suspended", "revoked"},
      "inspection_scheduled": {"inspection_complete", "issued"},
      "inspection_complete": {"completed", "inspection_scheduled"},
      "completed": {"expired"},
      "suspended": {"issued", "revoked"},
      ...
  }
  def transition(permit, new_status, user, reason) -> None
  ```
  Invalid transitions raise `InvalidStatusTransition`. Every transition writes to `permit_status_history` and emits a domain event.
- Permit numbering: per-jurisdiction sequence with configurable pattern (e.g., `BLDG-{YYYY}-{seq:06d}`) stored on `jurisdictions.config`. Service: `services/permits/numbering.py`.
- `form_data` is validated server-side against the permit type's JSON Schema before persisting.
- Endpoints:
  - `POST /api/v1/permits` — create draft (no permit_num yet).
  - `PATCH /api/v1/permits/{id}` — update draft form_data.
  - `POST /api/v1/permits/{id}/submit` — validates completeness, assigns permit_num, instantiates workflow, transitions to `submitted`.
  - `POST /api/v1/permits/{id}/transitions` — explicit status transition with reason.
  - `GET /api/v1/permits` — paginated list with filters (`status`, `permit_type_id`, `assigned_reviewer_id`, `applicant_id`, `parcel_id`, `applied_date_from`, `applied_date_to`, `q` full-text).
  - `GET /api/v1/permits/{id}` — full detail with related entities.

**Testing**:
- `Unit: transition(draft → issued) → InvalidStatusTransition`
- `Unit: transition(draft → submitted) writes permit_status_history with previous_status='draft'`
- `Unit: permit numbering generates unique sequential numbers under concurrent contention (100 concurrent submits in test → 100 distinct numbers)`
- `Integration: submit permit missing required form field → 422 with field paths`
- `Integration: submit permit with valid data → permit_num issued, workflow_instance created, first step in_progress`
- `API: filter combinations all return correct subsets (matrix test of 20 combinations)`
- `E2E: applicant fills 5-page form, submits, sees permit_num and "Submitted" status`

#### 3.2 — Workflow runtime engine

**What**: Implement step advancement, parallel branching, conditional routing, and SLA tracking.

**Design**:
- `services/workflows/engine.py` exposes:
  ```python
  def complete_step(step_instance_id: UUID, user_id: UUID, decision: Literal["approve","deny","revise","hold"], notes: str | None) -> AdvanceResult
  ```
  - Marks current step `completed` with decision.
  - Evaluates dependencies for downstream steps: when all `finish_to_start` deps complete with allowed decisions, downstream steps move from `pending` to `in_progress` and `started_at` set.
  - On `deny` → permit transitions to `denied` and active steps cancelled.
  - On `revise` → permit transitions to `revisions_requested`; applicant notified; on resubmit, returned steps move back to `in_progress`.
  - Computes `due_date` from `started_at + sla_days` accounting for jurisdiction business calendar (configured per jurisdiction in `config.business_days`).
- Domain events emitted to in-process bus + Celery for side effects: `PermitSubmitted`, `StepAssigned`, `StepCompleted`, `WorkflowCompleted`, `SLABreached`.
- Celery beat job `check_sla_breaches` runs every 15 minutes, marks overdue steps, emits `SLABreached`, increments Prometheus counter `permit_workflow_sla_breach_total`.

**Testing**:
- `Unit: complete_step on intake (no deps) → next step transitions to in_progress`
- `Unit: complete_step with deny decision → permit moves to 'denied', all in_progress steps cancelled`
- `Unit: parallel steps (zoning + building) both complete → downstream fee_assessment starts`
- `Unit: SLA check on step started 5 days ago with sla_days=3 → marked breached`
- `Integration: full residential workflow runs to completion → permit status = 'completed', workflow_instance status = 'completed'`
- `Concurrency: two reviewers complete the same step simultaneously → optimistic lock prevents double-advance`

#### 3.3 — Document upload and management

**What**: Implement `documents` table, S3-backed upload pipeline with virus scanning hook, versioning, and metadata.

**Design**:
- Pre-signed upload pattern:
  - `POST /api/v1/permits/{id}/documents/upload-url` → returns pre-signed PUT URL with constraints (`document_type`, `max_file_size_mb`, `accepted_formats`).
  - Client PUTs directly to blob store.
  - `POST /api/v1/permits/{id}/documents/confirm` → server verifies object exists, runs MIME sniff + size check, creates `documents` row, enqueues `scan_document` Celery task.
- `tasks/document_tasks.py:scan_document` calls configured virus scanner (ClamAV via clamd by default, plug-in interface for ENS/Cloud AV).
- Versioning: uploading same `document_type` flips previous row's `is_current_version=false` and sets `previous_version_id` on new row.
- `BlobStore` interface: `put`, `get`, `presign_put`, `presign_get`, `delete`, `exists`.

**Testing**:
- `Unit: BlobStore.presign_put returns URL with TTL ≤ 15 min and required content-type header`
- `Unit: upload exceeding max_file_size_mb in confirm → rejected, blob deleted`
- `Integration (MinIO): full upload-confirm-download cycle works`
- `Integration: virus scanner returning INFECTED → document marked quarantined, applicant notified`
- `Integration: upload v2 of site_plan → v1.is_current_version=false, v2 linked to v1`
- `E2E: applicant uploads PDF, sees it listed; uploads new version, sees both with v2 marked current`

#### 3.4 — Notifications service (email + SMS)

**What**: Implement `notifications` table, channel adapters, and template-rendered messages triggered by domain events.

**Design**:
- `integrations/email/base.py` and `integrations/sms/base.py` define provider-agnostic interfaces.
- Templates in `services/notifications/templates/` as Jinja2 files plus a YAML manifest binding event types to templates:
  ```yaml
  permit_submitted:
    channels: [email, in_app]
    email_template: permit_submitted_email.html.j2
    sms_template: null
    subject: "Permit {{ permit.permit_num }} received"
  ```
- Celery task `send_notification` handles retry with exponential backoff (max 5 attempts, jitter), updates `notifications.status`.
- Recipient preferences honoured (`user.notification_preferences` JSONB column added in this task).
- All notification content is i18n-localised; default locale per recipient, fallback to jurisdiction locale.

**Testing**:
- `Unit: render template with permit context → produces expected HTML/text`
- `Unit: notification with channel=sms but recipient lacks phone → status='failed', error captured`
- `Integration (mocked SMTP): permit_submitted event → email sent within 5s, status='delivered' after webhook`
- `Integration: third send attempt failure → status='failed', alert emitted`
- `E2E: applicant submits permit → receives confirmation email visible in MailHog`

#### 3.5 — Applicant portal: permit submission flow

**What**: Build the applicant-facing UI for browsing permit types, completing dynamic forms, uploading documents, and tracking status.

**Design**:
- Routes in `apps/portal/src/app`:
  - `/` landing + eligibility wizard entry (wizard built in Phase 5).
  - `/apply` → select permit type.
  - `/apply/{permitTypeId}` → multi-step dynamic form using `packages/forms` renderer driven by the JSON Schema from 2.2.
  - `/permits` → list of user's permits.
  - `/permits/{id}` → status timeline, documents, fees (read-only Phase 3), messages.
- `packages/forms/DynamicForm` walks the JSON Schema, supports conditional fields via `conditional_on`, integrates parcel picker for `field_type: parcel_lookup`.
- Status timeline component visualises workflow steps with `pending`/`in_progress`/`completed`/`returned` states.
- All inputs labelled, focus rings preserved, error messages associated via `aria-describedby`.

**Testing**:
- `E2E (Playwright): unauthenticated user clicks "Apply" → prompted to log in → returns to form`
- `E2E: complete a 4-step residential permit form → upload site_plan PDF → submit → see permit number`
- `E2E: conditional field appears/hides as parent field changes`
- `A11y: every step of the apply flow passes axe-core checks with zero violations`
- `A11y: keyboard-only user can complete the entire form (Playwright with no mouse interactions)`

#### 3.6 — Staff console: review queue and step action UI

**What**: Build the staff-facing UI for the reviewer queue, permit detail, and step action (approve / deny / revise) controls.

**Design**:
- `apps/staff/src/app`:
  - `/dashboard` → KPI cards (assigned to me, overdue, recent submissions).
  - `/queue` → filterable, sortable table (TanStack Table) of `workflow_step_instances` assigned to the user.
  - `/permits/{id}` → full permit detail with tabs (Details, Documents, Workflow, Fees, Inspections, History).
  - `StepActionDialog` collects decision + notes, calls `complete_step` API, optimistic UI update, toast on success.
- WebSocket channel `/ws/jurisdiction/{id}` (FastAPI websockets) broadcasts step state changes so multiple reviewers see updates in real time.

**Testing**:
- `E2E: reviewer opens queue, filters by 'overdue' → only SLA-breached items visible`
- `E2E: reviewer approves step → workflow advances → permit appears in next reviewer's queue`
- `E2E: reviewer A and B view same permit → A approves → B sees status update without refresh (websocket)`
- `A11y: queue table is screen-reader friendly (proper headers, row labels, focus management)`

---

## Phase 4: Fees, Payments, and Inspection Scheduling

### Purpose
Add the operational components that close the permit lifecycle loop: fee assessment, online payment processing, inspection scheduling, mobile inspector workflows, and re-inspection management. After Phase 4 a permit can be paid for and inspected end-to-end.

### Tasks

#### 4.1 — Permit fee assessment and invoicing

**What**: Wire up `permit_fees` creation triggered by workflow events (fee_assessment step) and expose fee summary endpoints.

**Design**:
- Event handler on `WorkflowStepStarted(step_type='payment')`:
  ```python
  def assess_fees(permit_id: UUID) -> list[PermitFee]
  ```
  Resolves applicable `fee_schedule`, calls `calculate_fees`, persists `permit_fees` rows with status `invoiced`, emits `FeesAssessed` event triggering notification.
- Endpoints:
  - `GET /api/v1/permits/{id}/fees` → list with totals.
  - `POST /api/v1/permits/{id}/fees/waive` (supervisor only) → mark fee waived with reason, audit logged.
  - `POST /api/v1/permits/{id}/fees/adjust` (supervisor only) → manual line addition.

**Testing**:
- `Unit: assess_fees idempotent (calling twice yields same fees, not duplicated)`
- `Integration: payment workflow step triggers fee assessment → portal shows fees due`
- `API: non-supervisor calls /waive → 403`

#### 4.2 — Stripe payment integration

**What**: Implement Stripe PaymentIntent flow, webhook handling, and `payments` + `payment_fee_allocations` recording.

**Design**:
- `integrations/payments/stripe.py` implements `PaymentProcessor`:
  ```python
  class PaymentProcessor(Protocol):
      def create_intent(self, amount: Decimal, currency: str, metadata: dict) -> IntentResult
      def verify_webhook(self, payload: bytes, signature: str) -> WebhookEvent
      def refund(self, processor_txn_id: str, amount: Decimal | None) -> RefundResult
  ```
- Endpoints:
  - `POST /api/v1/permits/{id}/payments/intent` → body lists `permit_fee_ids`, returns Stripe client secret.
  - `POST /api/v1/webhooks/stripe` → verifies signature using stripe-signature header, processes `payment_intent.succeeded` → creates `payments` row + `payment_fee_allocations`, transitions matching fees to `paid`, advances workflow payment step.
  - Idempotency via Redis key `webhook:stripe:{event.id}` 24h TTL.
- PCI scope: card data only enters Stripe Elements on client; webhook is the only ingress for transaction confirmation; we never store PAN.

**Testing**:
- `Unit: verify_webhook with tampered signature → SignatureError`
- `Unit: duplicate webhook event id → second processing is no-op`
- `Integration (Stripe test mode): create intent → simulate confirm → webhook fires → payment + allocations persisted`
- `Integration: partial payment (fee_ids subset) → only those fees marked paid, others remain due`
- `E2E (Stripe Elements + test card 4242): applicant completes payment → status updates → workflow advances`

#### 4.3 — Inspection types, checklists, and scheduling primitives

**What**: Implement `inspection_types`, `inspection_checklists`, `permit_type_inspections`, `inspections`, and basic scheduling endpoints.

**Design**:
- When permit transitions to `issued`, a Celery task creates `inspections` rows (status `requested`) for each `permit_type_inspections` entry.
- Endpoints:
  - `GET /api/v1/inspections` — filters: status, inspector_id, scheduled_date range, jurisdiction, permit_id.
  - `POST /api/v1/inspections/{id}/schedule` — body: scheduled_date, time_start, time_end, inspector_id. Validates inspector availability (no overlapping inspections), transitions to `scheduled`.
  - `POST /api/v1/inspections/{id}/reschedule` — same body, must include reason.
  - `POST /api/v1/inspections/{id}/cancel` — with reason.
- Inspector availability calendar: derived from `users.work_schedule` JSONB column (added here) + existing assigned inspections.

**Testing**:
- `Unit: schedule inspection overlapping with inspector's existing inspection → 409 Conflict`
- `Unit: schedule outside inspector work_schedule → 422 with explanation`
- `Integration: permit transitions to issued → inspection rows created automatically`

#### 4.4 — Inspector PWA — offline-first capture

**What**: Build the inspector mobile PWA with offline inspection list, checklist execution, photo capture, GPS stamp, and sync queue.

**Design**:
- `apps/inspector` is a Next.js PWA with `next-pwa` configuration generating a service worker.
- Offline storage in IndexedDB (Dexie.js) tables: `inspections`, `checklist_results`, `pending_uploads`, `sync_queue`.
- On launch, sync downloads assigned inspections for next 7 days plus checklist templates.
- Inspection screen:
  - Header: address, permit_num, GIS map pin (cached tiles).
  - Checklist: each item is pass/fail/na with notes.
  - Photo capture: writes to IndexedDB blob store; upload deferred to sync.
  - Submit: result + notes + checklist_results queued.
- Sync engine:
  - When online, drains `sync_queue` FIFO: API calls + photo PUTs to pre-signed URLs.
  - Conflict resolution: server is authoritative; if `inspections.updated_at` on server is newer, prompt inspector to merge.
- Re-inspection: failed inspection's "Schedule re-inspection" button creates a new inspection with `parent_inspection_id` link.

**Testing**:
- `Unit (frontend): sync queue serialises inspection result + photos into ordered ops`
- `Integration (Playwright with offline emulation): record inspection offline → go online → all data persists to backend including photos`
- `Integration: re-inspection chain (original failed → re-inspection passed) visible on permit timeline`
- `E2E: inspector uses PWA on simulated mobile (Playwright `--device='iPhone 14'`) end-to-end`
- `A11y: high-contrast mode and large text settings preserved on inspection screens (outdoor visibility)`

#### 4.5 — Inspection results, completion, and permit close-out

**What**: Implement inspection result submission, checklist results persistence, attachment confirmation, and permit close-out logic.

**Design**:
- Endpoints:
  - `POST /api/v1/inspections/{id}/result` — body: result, notes, checklist_results, attachment_ids. Transitions inspection status, writes `inspection_checklist_results`.
  - `POST /api/v1/inspections/{id}/attachments/upload-url` — pre-signed URL for photo upload.
  - When all required inspections for permit are `passed`, workflow step `final_inspection` completes → permit transitions to `inspection_complete` → `completed`.
  - Failed critical checklist item auto-fails the inspection.
- Real-time push: `WorkflowEvents` channel notifies applicant on inspection result via in-app notification.

**Testing**:
- `Unit: critical checklist item failed → inspection.result = 'fail' regardless of other items`
- `Unit: all required inspections passed → permit auto-transitions to completed`
- `Integration: result submitted with 3 photos → all attachments visible on permit detail`
- `E2E: inspector submits passing inspection → applicant sees "Inspection Passed" notification immediately`

---

## Phase 5: AI Layer — Completeness, Eligibility Assistant, Plan Review Pre-Screening

### Purpose
This is the differentiating phase. Phase 5 introduces the AI-native capabilities that distinguish this platform from incumbents: pre-submission completeness checking on uploaded documents, a conversational eligibility assistant guiding applicants to the correct permit pathway, and AI plan review pre-screening flagging likely code issues for human reviewers. All AI features run through a pluggable provider so jurisdictions can choose Anthropic, OpenAI, Azure OpenAI, or self-hosted models.

### Tasks

#### 5.1 — LLM provider abstraction and prompt library

**What**: Implement `LLMProvider` interface, Anthropic + OpenAI + local (Ollama) adapters, versioned prompt template loader, and cost/latency observability.

**Design**:
- `ai/providers/base.py`:
  ```python
  class LLMProvider(Protocol):
      async def complete(self, system: str, messages: list[Message], *, model: str, temperature: float, max_tokens: int, response_schema: type[BaseModel] | None = None) -> LLMResponse
      async def embed(self, texts: list[str], *, model: str) -> list[list[float]]
      async def stream(self, ...) -> AsyncIterator[str]
  ```
- Prompt templates stored in `ai/prompts/` as YAML with frontmatter:
  ```yaml
  ---
  id: completeness_check
  version: 1
  default_model: claude-haiku-4
  max_tokens: 2000
  ---
  system: |
    You are an expert plan reviewer for a US municipal building department...
  user: |
    PERMIT TYPE: {{ permit_type.name }}
    REQUIRED DOCUMENTS: {{ required_docs | join(', ') }}
    UPLOADED DOCUMENTS: ...
  ```
- `PromptRegistry.load(id, version)` returns a validated `PromptTemplate`. All runtime calls record `prompt_id`, `prompt_version`, `model`, `input_tokens`, `output_tokens`, `cost_usd`, `latency_ms` to `ai_invocations` table.
- Per-jurisdiction config: `jurisdictions.config.ai = { "provider": "anthropic", "default_model": "claude-opus-4", "max_monthly_spend_usd": 5000 }`. Soft budget alarms, hard budget caps enforced before each call.

**Testing**:
- `Unit: prompt template Jinja2 rendering with all required vars → expected string`
- `Unit: budget cap enforcement → BudgetExceededError before provider call`
- `Integration (mocked Anthropic): structured response with response_schema validates as Pydantic model`
- `Integration (mocked): retries on 429 with exponential backoff`
- `Integration: ai_invocations row written per call with accurate token counts`

#### 5.2 — AI document completeness checker

**What**: When applicant uploads documents, an async task analyses each document, classifies it (matches against required `document_type`), and computes a permit-level `ai_completeness_score` with actionable feedback.

**Design**:
- Triggered by `DocumentUploaded` event. Task `check_completeness(permit_id)`:
  1. Loads permit, permit_type, required_documents, all current documents.
  2. For each document, calls Claude with the doc PDF (rendered via PyMuPDF) + classification prompt (`prompts/document_classify.yaml`).
  3. Compares classified types to required types, computes coverage.
  4. For each uploaded document, runs a content-quality check (legibility, presence of expected sections via prompt `prompts/document_quality.yaml`).
  5. Aggregates score: `(documents_present / documents_required) * 0.6 + avg(document_quality_scores) * 0.4`.
  6. Updates `permits.ai_completeness_score`, `permits.ai_completeness_checked_at`, writes structured findings to `permit_ai_findings` table (added in this task migration).
  7. If score < 0.7, emits notification to applicant with specific issues.
- Endpoint: `GET /api/v1/permits/{id}/ai-completeness` returns score + findings.
- Reviewer-facing UI: `permits/{id}` detail page shows a "Completeness" panel with checklist + score and "Re-run check" button.

**Testing**:
- `Unit (mocked LLM): classification mapping logic with edge cases (no matches, multiple matches per doc type)`
- `Fixture: golden permit with 5 known documents → completeness score within tolerance, expected findings present`
- `Integration: upload triggers task → score persisted within 60s`
- `Integration: missing required doc → finding mentions document_type by name`
- `Cost test: full check on 5-document permit using haiku model → < $0.05 (verified against ai_invocations table)`

#### 5.3 — Eligibility assistant (conversational intake guidance)

**What**: A chat-style assistant on the portal landing page that asks plain-language questions about a project and recommends one or more permit types.

**Design**:
- Conversation API:
  - `POST /api/v1/ai/eligibility/session` → creates an `eligibility_sessions` row, returns `session_id`.
  - `POST /api/v1/ai/eligibility/{session_id}/message` → body `{ user_message }`. Server appends to history, calls LLM with system prompt (tool-using):
    ```
    You are a permit eligibility assistant for {{ jurisdiction.name }}. Available permit types: {{ permit_type_corpus }}. Ask clarifying questions about scope, location, structure type, and value, then recommend permit types with confidence.
    ```
    Tools exposed via JSON-mode:
    - `lookup_zoning(address)` → returns zoning_code from PostGIS lookup.
    - `list_permit_types(category)` → returns matching types.
    - `recommend_permit_types(recommendations: list[{permit_type_id, reason, confidence}])` → terminal tool, ends session and returns structured recommendations.
- `eligibility_sessions` table: id, user_id (nullable for anonymous), jurisdiction_id, transcript JSONB, recommendations JSONB, status, started_at, ended_at.
- Frontend chat UI in `apps/portal/src/app/start`:
  - Message bubbles, typing indicator, "Apply for this permit" CTA on recommended types.
  - Conversation persists across page refreshes via session_id in URL.

**Testing**:
- `Unit: tool router dispatches lookup_zoning to PostGIS service`
- `Unit: recommendation tool produces validated structured output`
- `Integration (mocked LLM): scripted conversation about "adding a deck" → recommendation includes "residential addition" permit type with ≥ 0.8 confidence`
- `Integration: zoning lookup tool used when user mentions an address`
- `E2E: applicant chats with assistant, accepts recommendation, lands on the apply page with permit type preselected`
- `A11y: chat UI announces new messages to screen readers via aria-live=polite`

#### 5.4 — Plan review pre-screening (vision-based)

**What**: When a plan PDF is uploaded for a permit requiring plan review, run an AI pre-screen extracting key building parameters (egress widths, occupancy load, fire-resistance ratings, etc.) and flagging likely IBC violations for human reviewers.

**Design**:
- Triggered when document with `document_type in ('site_plan', 'floor_plan', 'structural_drawing')` uploaded for a permit_type with `requires_plan_review=true`.
- Task `prescreen_plan(document_id)`:
  1. Load PDF, render each page to 2048px PNG via PyMuPDF.
  2. For each page, call Claude vision (`claude-opus-4`) with prompt `prompts/plan_review_extract.yaml`. Return structured `PlanExtractionResult`:
     ```python
     class PlanExtractionResult(BaseModel):
         page: int
         drawing_type: Literal["site","floor","elevation","section","detail","schedule","other"]
         scale: str | None
         rooms: list[Room]
         doors: list[Door]              # width, swing, hardware
         stairs: list[Stair]            # rise, run, width, handrails
         occupancy_indicators: list[OccupancyHint]
         fire_separation_walls: list[FireWallHint]
         notes: list[str]
     ```
  3. Run rule-based checks against extracted data using `services/plan_review/rules/`:
     - `egress_door_min_width` (IBC 1010.1.1): min 32" clear width.
     - `egress_path_count` (IBC 1006.2): occupancy-dependent.
     - `stair_rise_run` (IBC 1011.5): max 7" rise, min 11" run.
     - `corridor_width` (IBC 1020.2): min 44" (or 36" for low occupancy).
     - Additional rules added incrementally.
  4. Each rule produces `PlanReviewFinding` with severity (`info`, `warning`, `violation`), IBC code reference, page, location.
  5. Persist to `plan_review_ai_findings` table.
  6. Auto-create `plan_review_comments` for severity ≥ `warning`, attributed to system user, status `open`, with `code_reference`.
- Endpoint: `GET /api/v1/permits/{id}/plan-review/ai-findings` returns grouped findings.
- Reviewer UI: in plan viewer (built in 5.5), AI findings appear as pre-populated markers the reviewer can confirm, dismiss, or modify.

**Testing**:
- `Unit: egress_door_min_width rule with 30" door → violation with IBC 1010.1.1 reference`
- `Unit: stair_rise_run rule with valid dimensions → no finding`
- `Fixture: golden floor plan PDF with known violations → prescreen produces expected findings (allow 1 false-negative tolerance to account for vision drift)`
- `Integration: upload triggers task → findings appear within 5 minutes`
- `Cost guard: prescreen on 10-page plan capped at $2.00, configurable per jurisdiction`

#### 5.5 — Plan viewer with markup tooling

**What**: Build a browser-based plan viewer using pdf.js with an annotation canvas, supporting AI findings, reviewer markups, status tracking, and BCF export.

**Design**:
- `packages/plan-viewer` React component:
  - Renders PDF pages via pdf.js at responsive scale.
  - Annotation layer: SVG overlay storing markups in screen coordinates → converted to PDF page coordinates for persistence.
  - Toolbar: select, callout, rectangle, freehand, text, measurement.
  - Side panel: list of comments grouped by status; click jumps to marker.
  - AI findings shown with a "robot" icon; reviewer can "Confirm" (converts to standard comment) or "Dismiss" (logs decision).
- Persistence: each markup → POST `/api/v1/documents/{id}/comments` with `location_x`, `location_y`, `page_number`, `comment_text`, `code_reference`.
- BCF export: `GET /api/v1/permits/{id}/plan-review/bcf` returns a BCF 3.0 zip (XML topics + viewpoints) generated via `services/plan_review/bcf.py`.

**Testing**:
- `E2E: reviewer opens 10-page PDF → adds 3 comments → reloads → comments persisted in correct positions`
- `E2E: reviewer confirms an AI finding → it converts to a standard comment with original code_reference preserved`
- `Integration: BCF export → valid bcf zip readable by Bluebeam Revu (validated against bcf-schema)`
- `A11y: annotation list panel is keyboard-navigable; comment text reachable without mouse`

---

## Phase 6: Licensing, Code Enforcement, Analytics, Open Data Export

### Purpose
Round out the platform with business/contractor licensing lifecycle, code enforcement case management with violation notices, operational analytics dashboards, and standards-compliant open data export (BLDS) and APIs (Open311). After Phase 6 the platform covers the full feature set called out as "should-have" in features.md.

### Tasks

#### 6.1 — Licence lifecycle (business, contractor, professional)

**What**: Implement `licence_types`, `licences`, the licence application workflow (reusing the permit workflow engine), renewal scheduling, and suspension/revocation actions.

**Design**:
- Licences reuse the same workflow engine: a licence has a `workflow_instance` just like a permit. The difference is the entity surfaces.
- Endpoints:
  - `POST /api/v1/licences/applications` — submit licence application (similar shape to permit submit).
  - `GET /api/v1/licences` — list with filters (status, expiry range, type).
  - `POST /api/v1/licences/{id}/renew` — submit renewal application (creates new linked licence app).
  - `POST /api/v1/licences/{id}/suspend` — admin only, with reason and effective_date.
  - `POST /api/v1/licences/{id}/revoke` — admin only.
- Celery beat job `licence_renewal_reminders` (daily): for licences expiring in 60/30/14/7 days, enqueue notification.
- Celery beat job `licence_expiry_processor` (daily): expired licences transition to `expired`, related users notified, audit logged.

**Testing**:
- `Unit: renew on active licence → new application created, original remains active until issued`
- `Integration: licence expires → status auto-updates → owner notified`
- `Integration: suspend licence → cannot apply for related permits until reinstated`

#### 6.2 — Code enforcement cases and violation notices

**What**: Implement `code_enforcement_cases`, `violation_notices`, case lifecycle, violation notice generation (PDF), and an anomaly-detection job linking expired licences to active business operations.

**Design**:
- Endpoints under `/api/v1/code-enforcement/`:
  - `POST /cases` — manual case creation by staff or Open311 webhook.
  - `GET /cases` — filterable list.
  - `GET /cases/{id}` — detail with violations, notices, related permit/licence.
  - `POST /cases/{id}/notices` — issue violation notice, generates PDF via `services/code_enforcement/notice_pdf.py` (WeasyPrint), enqueues delivery (email + postal mail integration optional).
  - `POST /cases/{id}/transitions` — case status transitions.
- Anomaly detection task `detect_licence_violations` (daily):
  - For each `code_enforcement_cases` source `ai_detected`, run pattern checks: e.g., business operating without active licence (signals from Open311 reports, recent permit applications by an unlicensed contractor).
  - Creates draft cases with `source='ai_detected'`, `assigned_officer_id` defaulting to district officer.

**Testing**:
- `Unit: notice PDF generation produces searchable PDF/A with expected fields populated`
- `Integration: Open311 webhook reporting violation → case created automatically`
- `Integration: contractor with expired licence applies for permit → anomaly job creates case linked to both`

#### 6.3 — Analytics dashboards

**What**: Build the staff analytics surface with processing-time, workload, volume, and bottleneck charts.

**Design**:
- Materialised views refreshed every 15 minutes (Postgres):
  - `mv_permit_cycle_times` — average time from submitted → issued by permit_type, jurisdiction, month.
  - `mv_inspector_workload` — inspections per inspector per week with avg duration.
  - `mv_step_bottlenecks` — average time-in-step per workflow_template_step.
  - `mv_application_volume` — daily/weekly/monthly counts by permit_type and status.
- API:
  - `GET /api/v1/analytics/cycle-times?from=&to=&permit_type_id=` returns structured time series.
  - `GET /api/v1/analytics/inspector-workload`
  - `GET /api/v1/analytics/bottlenecks`
  - `GET /api/v1/analytics/volume`
- Frontend dashboards in `apps/staff/src/app/analytics` using Recharts or visx.
- Export: each chart has CSV/PNG export action.

**Testing**:
- `Unit: cycle time aggregation handles permits still in flight (excluded)`
- `Integration: materialised view refresh keeps data within freshness target (< 20 minutes)`
- `E2E: staff user filters cycle time chart by permit_type → chart updates with correct data`
- `A11y: charts have accessible alternative table view (axe + manual)`

#### 6.4 — BLDS open data export and public API

**What**: Implement automated BLDS-compliant export (permits.csv, inspections.csv, contractors.csv) published on a configurable schedule, and a public read-only API.

**Design**:
- `exports/blds.py` produces three CSVs matching the BLDS schema field names exactly (we aligned column names in Phase 1):
  ```python
  def generate_blds_bundle(jurisdiction_id: UUID, since: date | None = None) -> BLDSBundle
  ```
  Output: zip with `permits.csv`, `inspections.csv`, `contractors.csv` + `metadata.json` (jurisdiction info, last_updated).
- Celery beat job `publish_blds_daily` (configurable per jurisdiction): generates bundle, uploads to a public blob bucket, updates `public_data_publications` table with URL.
- Public endpoints (no auth, rate-limited 60 req/min per IP):
  - `GET /api/v1/public/{jurisdiction_id}/permits` — paginated, filterable, JSON (also `Accept: text/csv` returns BLDS-formatted CSV stream).
  - `GET /api/v1/public/{jurisdiction_id}/inspections`
  - `GET /api/v1/public/{jurisdiction_id}/blds/latest` — 302 redirects to latest bundle URL.
  - `GET /api/v1/public/datasets/index` — JSON index of all published datasets (DCAT-compatible).
- Privacy filter: applicant_email, applicant_phone, latitude/longitude rounded to 4 decimals for public output (jurisdiction-configurable).

**Testing**:
- `Fixture: golden jurisdiction with 100 permits → generated permits.csv byte-identical to expected golden file (modulo timestamps)`
- `Unit: BLDS field mapping correct for all permit statuses`
- `Integration: public API returns rows scoped to jurisdiction, never exposes other tenants`
- `Integration: rate limit kicks in at 61 req/min`
- `Validation: generated CSV passes BLDS schema validation (csvschema)`

#### 6.5 — Open311 endpoint

**What**: Implement the Open311 GeoReport v2 API for service-request integration.

**Design**:
- Endpoints under `/api/v1/public/{jurisdiction_id}/open311/`:
  - `GET /services.{format}` — list service types (format = `json` | `xml`).
  - `GET /services/{service_code}.{format}` — service detail.
  - `POST /requests.{format}` — create service request (e.g., a code enforcement complaint). Maps to `service_requests` row; if matches code_enforcement service type, creates a case.
  - `GET /requests.{format}` — paginated list, filterable.
  - `GET /requests/{id}.{format}` — detail.
- Returns both JSON and XML per Open311 spec (use `dicttoxml`).

**Testing**:
- `Integration: POST a request → service_request row created, mapped case visible to staff`
- `Schema: response validates against Open311 GeoReport v2 schema`

---

## Phase 7: Hardening — Accessibility, Security, Performance, Localisation

### Purpose
Make the platform production-ready for government deployment. Phase 7 systematically addresses non-functional requirements: WCAG 2.1 AA audit + fixes, security review (NIST SP 800-53 baseline mapping, OWASP ASVS), performance benchmarking and tuning, multi-language support, and FedRAMP control documentation.

### Tasks

#### 7.1 — Accessibility audit and remediation

**What**: Run a full WCAG 2.1 AA audit across all UI surfaces and remediate all violations.

**Design**:
- Automated CI gate: `pnpm test:a11y` runs Playwright against every public route with `@axe-core/playwright`, fails build on any violation tagged `wcag2a`, `wcag2aa`, `wcag21a`, `wcag21aa`.
- Manual audit checklist run by a contracted accessibility specialist covering the 50 WCAG 2.1 AA success criteria; findings tracked in `docs/accessibility/audit-2026.md`.
- Issues prioritised P1 (blocker for v1.0) / P2 (must fix before v1.1) and remediated.
- Specific deliverables:
  - All interactive components have visible focus rings and pass focus-order testing.
  - Colour contrast ≥ 4.5:1 (verified by axe + Stark plugin).
  - Forms have programmatic labels, error association, and recovery instructions.
  - Animations respect `prefers-reduced-motion`.
  - Tables in staff console have proper `<th scope>` and caption.
  - Plan viewer has keyboard alternative for annotation (T key to add comment at focused element).

**Testing**:
- `CI: axe-core full ruleset, zero violations across portal/staff/inspector routes`
- `Manual: VoiceOver + macOS Safari smoke test on 5 critical user journeys`
- `Manual: NVDA + Firefox Windows smoke test on same 5 journeys`
- `Manual: 200% zoom on portal application form → no horizontal scroll, all content reachable`

#### 7.2 — Security baseline (OWASP ASVS L2, NIST SP 800-53 mapping)

**What**: Conduct security review against OWASP ASVS Level 2 and document NIST SP 800-53 control implementation.

**Design**:
- Required controls implemented or verified:
  - Session: token rotation on refresh, idle timeout (30 min default), absolute timeout (12 h), revocation on logout, session fixation prevention.
  - Input validation: every API endpoint accepts only Pydantic-validated input; no raw SQL accepts user input.
  - SSRF prevention: outbound HTTP client (httpx) restricted from RFC 1918 / link-local destinations by default; allowlist for known integrations.
  - Secrets management: all secrets via environment or external secret manager (AWS Secrets Manager, Vault); no secrets in code, logs, or audit_log.
  - Headers: CSP (strict-default-src), HSTS (max-age=31536000), X-Frame-Options DENY, X-Content-Type-Options nosniff applied by middleware.
  - Rate limiting: per-IP (60 req/min anon, 600 req/min authenticated), per-user, per-endpoint configurable.
  - Audit logging covers all 800-53 AU-2 events (auth, auth changes, data export, admin actions).
- Document mapping in `docs/security/nist-800-53-mapping.md` listing each NIST control and the implementation reference (file + function).
- Dependency scanning in CI: `pip-audit`, `pnpm audit`, Trivy on container images.
- SAST: `semgrep --config=auto` in CI, blocking on high-severity findings.

**Testing**:
- `Unit: outbound HTTP to 169.254.169.254 → SSRFError`
- `Unit: log records redact known secret patterns (api keys, JWTs, card numbers)`
- `Integration: rate limit returns 429 with Retry-After header`
- `Integration: response includes all required security headers (validated by securityheaders.com-style checks)`
- `Pentest: external pentest report (annual) with no critical/high findings before v1.0 release`

#### 7.3 — Performance benchmarking and tuning

**What**: Establish performance SLOs, run load tests, and tune hot paths.

**Design**:
- SLO targets for typical mid-sized jurisdiction (50k permits/year):
  - p95 portal page load < 2.5 s (LCP).
  - p95 API response < 400 ms.
  - p99 < 1500 ms.
  - Inspection sync (50 records) < 5 s.
- Load test scripts in `tools/load-test/`:
  - `k6-portal-browse.js` — 200 VUs browsing.
  - `k6-permit-submit.js` — 50 VUs submitting permits.
  - `k6-public-blds.js` — 1000 VUs reading public API.
- Tuning: add missing indexes (driven by `pg_stat_statements`), tune `work_mem`, switch hot read paths to read replica.
- Continuous: golden-path traces captured in OpenTelemetry; per-endpoint p95 dashboards in Grafana.

**Testing**:
- `Load: k6-portal-browse for 10 min → p95 LCP < 2.5s, error rate < 0.1%`
- `Load: k6-permit-submit → no failed submits at 50 RPS sustained`
- `Regression: CI runs reduced k6 scenarios; PR fails if p95 regresses > 20% from baseline`

#### 7.4 — Internationalisation and multi-language

**What**: Add i18n infrastructure with English (US) baseline, Spanish (US), and the framework for jurisdictions to add additional locales.

**Design**:
- Backend: notification templates and validation messages stored per locale in `services/notifications/templates/{locale}/`.
- Frontend: next-intl with `messages/{locale}.json`. Default to `en-US`; user preference stored in `users.locale`.
- i18n key conventions: dot.namespaced, descriptive (`portal.apply.button.submit`); empty values surface in lint as missing.
- Right-to-left support deferred; Spanish (and future French, Vietnamese) only.
- Permit type and licence type names/descriptions: `translations` table (added Phase 1 mention realised here):
  ```sql
  CREATE TABLE translations (
      id UUID PRIMARY KEY,
      jurisdiction_id UUID NOT NULL,
      entity_type VARCHAR(50) NOT NULL,   -- 'permit_type.name', 'licence_type.name', ...
      entity_id UUID NOT NULL,
      locale VARCHAR(10) NOT NULL,
      content TEXT NOT NULL,
      UNIQUE(entity_type, entity_id, locale)
  );
  ```

**Testing**:
- `Unit: missing key in non-default locale falls back to default with logged warning`
- `Integration: switch portal locale → all UI strings update, server returns localised email`
- `E2E: Spanish-speaking applicant completes permit application → confirmation email in Spanish`

#### 7.5 — Deployment artefacts and operator documentation

**What**: Publish production-grade deployment options and operator documentation.

**Design**:
- `deploy/helm/permitting-platform/` Helm chart with values for HA Postgres, Redis Sentinel, S3, OIDC, secrets, and resource sizing presets (`small`, `medium`, `large`).
- `docker-compose.prod.yml` for single-tenant deployments with sensible defaults and TLS via Caddy.
- Operator docs in `docs/operators/`:
  - Installation (Helm + compose).
  - Backup/restore (pg_dump/pg_basebackup + blob backup).
  - Disaster recovery runbook with RTO/RPO targets.
  - Upgrade procedure (Alembic migrations, zero-downtime strategy).
  - SLO + alerting reference (Prometheus rules included).
  - Tenant onboarding (creating jurisdiction, importing parcels, seeding permit types).
- Terraform reference modules for AWS, Azure, GCP in `deploy/terraform/`.

**Testing**:
- `Integration: helm install on kind cluster → all pods healthy in <5 minutes, /healthz responds`
- `Integration: backup then restore on separate cluster → data integrity verified by checksum`
- `Manual: follow operator install guide on clean Ubuntu VM → working install in <60 min`

---

## Phase 8: Backlog Differentiators — Predictive Scheduling, BIM/IFC, Multi-Jurisdiction

### Purpose
Phase 8 lifts the platform from "table-stakes parity" to "category-leading". It implements the longer-horizon differentiators identified in `features.md` (predictive inspection routing, BIM/IFC intake, multi-jurisdiction federation). Each task is independently shippable; teams can pick the highest-priority ones.

### Tasks

#### 8.1 — Predictive inspection scheduling

**What**: ML-based optimisation that proposes daily inspector schedules minimising drive time while respecting SLAs and inspector specialisations.

**Design**:
- Inputs: requested inspections (with lat/lng), inspector locations, inspector specialisations, work_schedule, historical inspection durations.
- Algorithm: VRP solver via OR-Tools (`ortools.constraint_solver.routing_enums_pb2`). Constraints:
  - Each inspection assigned to qualified inspector.
  - Travel time + duration fits within inspector shift.
  - SLA priority weighting in objective.
- Endpoint: `POST /api/v1/inspections/optimise` body: `date_range`, `inspector_ids?`. Returns proposed schedule; staff supervisor reviews and applies.
- Travel time: OSRM (self-hosted) or pluggable provider (Google Maps, Mapbox).

**Testing**:
- `Unit: VRP with 20 inspections + 3 inspectors → all assigned, total drive < naive sequential`
- `Integration: optimise endpoint returns within 30s for 100 inspections`
- `Backtest: replay historical day → optimised schedule shows ≥ 15% drive-time reduction vs actual`

#### 8.2 — BIM/IFC intake and digital permit pathway

**What**: Accept IFC file uploads, parse with ifcopenshell, extract building parameters automatically, and run plan review rules against IFC data.

**Design**:
- `bim_submissions` table (referenced in standards alignment) stores IFC blob reference + extraction metadata.
- `services/bim/ifc_extract.py`:
  ```python
  class IFCExtraction(BaseModel):
      ifc_version: str             # IFC4, IFC4X3
      building: BuildingProperties # gross_area, height, storeys
      spaces: list[SpaceExtract]   # room name, area, occupancy
      doors: list[DoorExtract]
      stairs: list[StairExtract]
      walls: list[WallExtract]     # fire rating
  ```
- Same rules engine from 5.4 runs against IFC data (preferred when both PDF and IFC present).
- Endpoint: `POST /api/v1/permits/{id}/bim/upload-url` → pre-signed URL; `POST /api/v1/permits/{id}/bim/confirm` enqueues parsing.

**Testing**:
- `Unit: ifcopenshell extract on sample IFC4 file → expected properties present`
- `Fixture: golden IFC with known violations → rules engine produces expected findings`
- `Integration: upload IFC → extraction completes < 2 min for 50MB model`

#### 8.3 — Multi-jurisdiction federation

**What**: Cross-jurisdiction permit lookup, shared contractor licence verification, and shared parcel data federation.

**Design**:
- Federation registry table `federated_jurisdictions` with peer endpoint URL + trust token.
- `services/federation/contractor_check.py`:
  ```python
  def federated_contractor_lookup(licence_number: str, peer_jurisdictions: list[UUID]) -> list[FederatedContractor]
  ```
  Hits peer `/api/v1/public/contractors?licence_number=...` endpoints (or signed federation endpoints if mutual trust established).
- Parcel federation: shared GIS layer via OGC WFS standard exposed by participating jurisdictions.

**Testing**:
- `Integration: federation across 3 test jurisdictions → contractor visible in all three's lookups`
- `Security: peer with invalid trust token → 401`

#### 8.4 — Virtual inspection (video review)

**What**: Schedule live video inspections via WebRTC with recording, screenshot, and checklist completion in-call.

**Design**:
- WebRTC via LiveKit (self-hosted) or pluggable Twilio Video.
- New inspection type variant `virtual_inspection` with `meeting_url` field.
- Inspector launches inspection page with live video tile, applicant joins via portal link.
- Recording stored in blob storage, transcript via Whisper (optional).

**Testing**:
- `E2E: schedule virtual inspection → both parties join → checklist completed → result submitted with recording attached`

#### 8.5 — Multi-language AI translation

**What**: AI-assisted translation of permit type descriptions, notices, and reviewer comments into applicant's preferred language.

**Design**:
- On-demand translation endpoint: `POST /api/v1/translate` body `{text, target_locale, context}`. Caches by content-hash + locale.
- Reviewer comments offered with "View in my language" toggle on applicant portal.

**Testing**:
- `Integration: translate technical IBC reviewer comment to Spanish → meaning preserved (human-spot-checked golden set)`
- `Cache: same content + locale on second request → no provider call (verified via ai_invocations)`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Tenancy, Identity                    ─── prerequisite for all
    │
    ▼
Phase 2: Configuration Domain                             ─── parcels, permit types, fees, workflows
    │
    ▼
Phase 3: Permit Lifecycle                                 ─── requires Phase 2
    │
    ▼
Phase 4: Fees, Payments, Inspections                      ─── requires Phase 3
    │
    ├── Phase 5: AI Layer                                 ─── requires Phase 3 (can start when 3 stable; runs in parallel with Phase 4)
    │
    └── Phase 6: Licensing, Enforcement, Analytics, Export ── requires Phase 4 (analytics needs operational data)
            │
            ▼
        Phase 7: Hardening                                ─── requires Phases 1–6 feature-complete
            │
            ▼
        Phase 8: Backlog Differentiators                  ─── post-v1.0; each task independently shippable
```

**Concrete parallelism opportunities:**
- Phase 5 (AI Layer) tasks 5.1–5.3 can begin as soon as Phase 3 is stable; 5.4–5.5 require Phase 4.3 (inspections) since plan review can complete a step.
- Phase 6.4 (BLDS export) can begin in parallel with Phase 5 since it operates on Phase 3 data.
- Phase 7.4 (i18n) can begin as a parallel track from Phase 3 onward to avoid retrofitting strings.
- Phase 8 tasks are fully independent of each other; teams can pick any.

**Suggested release cadence:**
- **v0.5 alpha** (Phases 1–3): early-access for two pilot jurisdictions to validate core lifecycle.
- **v0.9 beta** (Phases 1–6 except 5.4/5.5): functionally complete for a small jurisdiction.
- **v1.0** (Phases 1–7 complete): production-ready, externally audited.
- **v1.1+** (Phase 8): differentiators added based on adopter demand.

---

## Definition of Done (per phase)

Every phase is considered complete only when all of the following hold:

1. All tasks in the phase implemented and merged to `main`.
2. All new and existing unit, integration, and E2E tests pass in CI.
3. `ruff check`, `ruff format --check`, `mypy --strict`, `eslint`, and `prettier --check` pass with zero findings.
4. Code coverage for new modules ≥ 80% (lines + branches).
5. All new API endpoints appear in the auto-generated `openapi.json`, are referenced in `docs/api/`, and pass schemathesis property tests against the spec.
6. All new database changes have forward and reverse Alembic migrations; `alembic upgrade head` followed by `alembic downgrade -1` followed by `alembic upgrade head` is clean.
7. All new tenant-scoped tables have RLS policies enabled and a regression test confirming cross-tenant isolation.
8. Docker images build for `linux/amd64` and `linux/arm64`; `docker-compose up --wait` brings the dev stack to healthy.
9. New configuration options documented in `.env.example` and `docs/operators/configuration.md` with rationale and defaults.
10. All new user-facing surfaces pass `axe-core` WCAG 2.1 AA checks in CI with zero violations and have at least one Storybook story per component.
11. All new logging emits structured JSON with request_id, jurisdiction_id, user_id correlation; new metrics registered in Prometheus catalogue.
12. Dependency licences of any new third-party packages are compatible with Apache 2.0 (no GPL contamination); recorded in `docs/legal/dependencies.md`.
13. Any new external integration (LLM provider, payment processor, email provider) is implemented behind the established abstraction and has both a real and a mock implementation suitable for tests.
14. CHANGELOG.md updated with phase summary; release notes drafted.
15. Phase demoed to maintainers against a checklist of acceptance scenarios specific to that phase's tasks.
