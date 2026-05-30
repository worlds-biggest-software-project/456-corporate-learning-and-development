# Corporate Learning & Development Platform — Phased Development Plan

> Project: `456-corporate-learning-and-development` · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files. The persistence layer is built on **Data Model Suggestion 3 (Hybrid Relational + Document, PostgreSQL JSONB)**: a single PostgreSQL engine gives self-hosted deployments low operational burden while delivering referential integrity for transactional LMS data and JSONB flexibility for the schema-fluid surfaces (xAPI extensions, SCORM runtime, heterogeneous skills taxonomies, tenant custom fields). Graph-style pathway optimisation (from Suggestion 4) is implemented later in-process using recursive CTEs over the skill prerequisite graph, avoiding a second datastore until scale demands it.

---

## Core Requirements Summary

**What it does.** An AI-native, open-source corporate L&D platform that fuses an LMS core (SCORM/xAPI/cmi5 hosting, enrolment, completion, compliance) with a skills intelligence layer (ESCO/O*NET-grounded taxonomy, role-to-skill gap analysis) and AI-driven content curation, pathway generation, and authoring — closing the gap between workforce capability and business strategy, and attributing learning to business KPIs.

**Who uses it.** L&D administrators (taxonomy, compliance rules, content), learners (assigned + recommended learning), managers (team gap heatmaps, coaching suggestions), and integration/IT owners (HRIS sync, SSO, API consumers).

**Key differentiators.** Open skills record grounded in free taxonomies (no proprietary ontology lock-in); AI skill-gap detection and pathway generation; AI authoring from documents/topics; ROI attribution linking learning to performance KPIs; MCP server exposing the platform inside enterprise LLMs.

**Deployment model.** Self-hosted / cloud / hybrid, Docker-first, multi-tenant (shared schema, `tenant_id` everywhere, PostgreSQL RLS).

**Standards that shape the design.** SCORM 1.2/2004, xAPI 1.0.3 (LRS), cmi5, LTI 1.3 (backlog), SCIM 2.0 (RFC 7643/7644), OAuth 2.0 (RFC 6749) + JWT (RFC 7519), OpenID Connect, SAML 2.0, OpenAPI 3.1, IEEE 1484.20.1 (competency definitions), Open Badges 3.0 / W3C VC 2.0 (credential export, backlog), MCP, OWASP Top 10, GDPR.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | **Python 3.12** | The product is LLM-heavy (gap detection, pathway generation, authoring, compliance Q-gen, ROI attribution). Python has the strongest LLM SDK ecosystem and statistical tooling (statsmodels/scipy) needed for ROI causal analysis. |
| API framework | **FastAPI** | Async-native (critical for fan-out LLM calls and SCIM/HRIS sync), generates OpenAPI 3.1 automatically — a hard requirement from `standards.md` — and integrates Pydantic v2 for request/response validation. |
| Data validation | **Pydantic v2** | Single source of truth for API schemas, JSONB payload validation (xAPI statements, SCORM CMI, tenant custom fields), and config. |
| ORM / DB access | **SQLAlchemy 2.0 (async) + Alembic** | Mature async ORM with first-class JSONB column mapping (required by the hybrid model); Alembic gives versioned, reversible migrations for the relational spine. |
| Database | **PostgreSQL 16** | Per Data Model Suggestion 3: relational integrity for enrolments/compliance/audit, JSONB + GIN for xAPI/SCORM/skills variance, `tsvector`+`pg_trgm` full-text, recursive CTEs for skill-graph traversal, RLS for tenant isolation. One engine = low self-hosted ops. |
| Task queue | **Celery + Redis** | Async workloads: SCIM/HRIS sync, LLM generation jobs, content-provider polling, compliance assignment sweeps, ROI report generation, notification dispatch. Redis doubles as broker + result backend + cache. |
| Cache / session | **Redis 7** | SCORM `suspend_data` live session state, dashboard read-model cache, rate-limit counters, Celery broker. |
| LLM access | **LiteLLM** (provider-agnostic gateway) | Differentiators require swappable models (OpenAI, Anthropic, Azure, self-hosted) for self-hosted sovereignty. LiteLLM gives one interface, retries, and cost tracking. |
| Object storage | **S3-compatible (boto3); MinIO for self-host** | SCORM/cmi5 packages, video assets, generated certificates. MinIO ships in `docker-compose` for zero external dependency. |
| Frontend | **Next.js 15 (React, TypeScript) + shadcn/ui + TanStack Query** | Three role-based portals (learner, manager, admin) with rich dashboards/heatmaps. Server components for fast initial loads; TanStack Query for live progress. SCORM player runs client-side via an iframe + JS API bridge. |
| Auth | **Authlib (OAuth2/OIDC) + python3-saml** | Implements OAuth 2.0 authorization-code + client-credentials flows for the API, OIDC and SAML 2.0 for enterprise SSO — exactly the matrix every incumbent exposes. |
| Containerisation | **Docker + docker-compose** | Self-hosted is a first-class deployment mode; compose bundles Postgres, Redis, MinIO, API, worker, and web. |
| Testing | **pytest + pytest-asyncio + testcontainers; Playwright (web)** | Unit + integration against ephemeral Postgres/Redis containers; Playwright for the SCORM-player and dashboard E2E flows. |
| Quality tools | **ruff (lint+format), mypy (strict), pre-commit** | Fast, single-tool lint/format; strict typing across the LLM and integration boundaries. |
| Package manager | **uv** (Python), **pnpm** (web) | Fast, lockfile-based, reproducible installs. |
| Key libraries | **httpx** (async HRIS/content/ESCO/O*NET clients), **scim2-filter-parser** (SCIM query parsing), **rdflib** (ESCO SKOS import), **statsmodels/scipy** (ROI attribution), **mcp** (Model Context Protocol server), **reportlab** (certificates/Open Badges) | Domain-specific needs drawn directly from `standards.md`. |

### Project Structure

```
corporate-ld-platform/
├── pyproject.toml                  # uv-managed, ruff/mypy/pytest config
├── alembic.ini
├── Dockerfile                      # API + worker image
├── docker-compose.yml              # postgres, redis, minio, api, worker, web
├── docker-compose.override.yml     # local dev overrides
├── .env.example
├── README.md
├── migrations/                     # Alembic versions
│   └── versions/
├── src/
│   └── ldp/
│       ├── __init__.py
│       ├── main.py                 # FastAPI app factory, router mounting
│       ├── config.py               # Pydantic Settings (env-driven)
│       ├── db/
│       │   ├── session.py          # async engine, session, RLS tenant context
│       │   ├── base.py             # declarative base, mixins (timestamps, tenant)
│       │   └── models/             # SQLAlchemy models, grouped by domain
│       │       ├── identity.py     # tenants, orgs, users, roles, idp
│       │       ├── skills.py       # frameworks, skills, job_roles, user_skills, gaps
│       │       ├── content.py      # providers, courses, modules, assets, assessments
│       │       ├── delivery.py     # pathways, enrolments, scorm_runtime, xapi, certs, compliance
│       │       ├── analytics.py    # learning_events, kpis, roi_reports
│       │       └── integration.py  # sync_log, notifications, audit_log
│       ├── schemas/                # Pydantic request/response + JSONB payload models
│       ├── api/
│       │   ├── deps.py             # auth, tenant, pagination, db-session deps
│       │   ├── errors.py           # RFC7807 problem+json handlers
│       │   └── v1/
│       │       ├── router.py
│       │       ├── auth.py  users.py  skills.py  roles.py  courses.py
│       │       ├── enrolments.py  pathways.py  compliance.py  analytics.py
│       │       ├── scorm.py  xapi.py  scim.py  webhooks.py
│       │       └── ...
│       ├── services/               # business logic (framework-agnostic, unit-testable)
│       │   ├── skills_service.py   gap_analysis.py  pathway_service.py
│       │   ├── enrolment_service.py  compliance_service.py
│       │   ├── scorm_runtime.py  lrs_service.py  roi_service.py
│       │   └── content_quality.py
│       ├── ai/
│       │   ├── client.py           # LiteLLM wrapper, retries, cost logging
│       │   ├── prompts/            # versioned prompt templates (jinja2)
│       │   ├── pathway_gen.py  authoring.py  compliance_qgen.py
│       │   ├── coaching.py  content_scoring.py  roi_attribution.py
│       │   └── schemas.py          # structured-output Pydantic models
│       ├── integrations/
│       │   ├── scim/               # SCIM 2.0 server (RFC 7643/7644)
│       │   ├── hris/               # workday.py, bamboohr.py (+ base.py)
│       │   ├── content/            # linkedin_learning.py, coursera.py (+ base.py)
│       │   ├── messaging/          # slack.py, teams.py
│       │   ├── taxonomy/           # esco.py, onet.py importers
│       │   └── sso/                # saml.py, oidc.py
│       ├── mcp/                    # MCP server exposing catalog/progress/certs
│       ├── workers/               # celery app + task modules
│       ├── security/              # rbac.py, oauth.py, crypto.py, audit.py
│       └── observability/         # logging, metrics, tracing
├── web/                            # Next.js app (learner/manager/admin portals)
│   ├── package.json
│   ├── app/
│   ├── components/
│   └── lib/
└── tests/
    ├── conftest.py                 # testcontainers fixtures, factory-boy factories
    ├── unit/
    ├── integration/
    ├── e2e/
    └── fixtures/                   # sample SCORM/cmi5 packages, ESCO subset, xAPI statements
```

The structure groups by concern (models / services / ai / integrations / api), not by phase — every phase adds files within existing packages without restructuring.

---

## Phase 1: Foundation, Multi-Tenancy & Identity

### Purpose
Establish the runnable skeleton: FastAPI app, async PostgreSQL with Alembic, configuration, the multi-tenant data spine, and the identity domain (tenants, organizations, users, roles). After this phase the platform can be deployed via `docker-compose up`, accept authenticated API calls, and enforce tenant isolation — the foundation every later domain depends on.

### Tasks

#### 1.1 — Project scaffold, config, and container stack

**What**: Bootstrap the repo, dependency management, FastAPI app factory, settings, and the Docker stack.

**Design**:
- `pyproject.toml` (uv) with deps from the table above; ruff + mypy(strict) + pytest config.
- `ldp/config.py` using Pydantic `BaseSettings`:
  ```python
  class Settings(BaseSettings):
      database_url: PostgresDsn
      redis_url: RedisUrl
      s3_endpoint: str; s3_bucket: str; s3_access_key: str; s3_secret_key: SecretStr
      jwt_secret: SecretStr; jwt_algorithm: str = "HS256"; access_token_ttl_s: int = 3600
      llm_model: str = "gpt-4o"; llm_api_base: str | None = None; llm_api_key: SecretStr | None = None
      cors_origins: list[str] = []
      environment: Literal["dev", "staging", "prod"] = "dev"
      model_config = SettingsConfigDict(env_prefix="LDP_", env_file=".env")
  ```
- `ldp/main.py` app factory `create_app() -> FastAPI`: mounts `/api/v1`, registers RFC 7807 error handlers, healthcheck `GET /healthz` → `{"status":"ok","db":bool,"redis":bool}`, and OpenAPI metadata (`title`, `version`, `openapi_version="3.1.0"`).
- `docker-compose.yml`: services `postgres:16`, `redis:7`, `minio`, `api`, `worker`, `web`; named volumes; healthchecks gating `depends_on`.

**Testing**:
- `Unit: Settings loads from env with LDP_ prefix → typed fields, secrets masked in repr`.
- `Integration: GET /healthz with DB+Redis up → 200, all flags true`.
- `Integration: GET /healthz with DB down → 503, db flag false`.
- `Smoke: docker-compose up → /healthz reachable on host port`.

#### 1.2 — Async DB layer, base models, and tenant context

**What**: Async SQLAlchemy engine/session, declarative base with shared mixins, and a request-scoped tenant context that drives RLS.

**Design**:
- `db/session.py`: `create_async_engine`, `async_sessionmaker`, FastAPI dependency `get_session()`. A `set_tenant(session, tenant_id)` helper issues `SET LOCAL app.current_tenant = :tid` so RLS policies (added in 1.3) filter rows.
- `db/base.py` mixins:
  ```python
  class TimestampMixin:
      created_at: Mapped[datetime] = mapped_column(server_default=func.now())
      updated_at: Mapped[datetime] = mapped_column(server_default=func.now(), onupdate=func.now())
  class TenantMixin:
      tenant_id: Mapped[UUID] = mapped_column(ForeignKey("tenants.tenant_id"), index=True)
  ```
- Alembic configured for async (`run_migrations_online` with async engine).

**Testing**:
- `Integration (testcontainers PG): create session, insert+select round-trips`.
- `Unit: TimestampMixin sets created_at/updated_at on flush`.
- `Integration: set_tenant issues SET LOCAL; second tenant cannot read first tenant's rows (verified after 1.3 RLS)`.

#### 1.3 — Identity domain models + RLS

**What**: Implement `tenants`, `organizations` (self-referential tree), `users` (self-referential `manager_id`), `roles`, `permissions`, `role_permissions`, `user_roles`, `identity_providers`, `user_identity_links` per Data Model Suggestion 3's identity domain, plus a JSONB `custom_fields` column on `users` and `tenants.settings_json`.

**Design**:
- SQLAlchemy models in `db/models/identity.py` mirroring the DDL (UUID PKs, the documented CHECK constraints expressed as Enums/`CheckConstraint`).
- RLS: Alembic migration enabling `ROW LEVEL SECURITY` on every tenant-scoped table with policy `USING (tenant_id = current_setting('app.current_tenant')::uuid)`.
- Seed migration: system permissions (`courses:read`, `users:write`, `reports:read`, …) and system roles (`admin`, `manager`, `learner`, `author`, `compliance_officer`).

**Testing**:
- `Unit: org tree — parent/child relationship resolves; cycle rejected by service guard`.
- `Integration: RLS — tenant A query returns only tenant A users even with no explicit WHERE`.
- `Integration: user_roles many-to-many; deleting a role cascades user_roles`.
- `Fixture: seed migration creates expected system roles/permissions`.

#### 1.4 — OAuth 2.0 / JWT auth and RBAC

**What**: Authentication (password + token issuance) and role/permission enforcement.

**Design**:
- `security/oauth.py`: OAuth 2.0 `password` and `client_credentials` token endpoints (`POST /api/v1/auth/token`) issuing JWTs (RFC 7519) with claims `sub`, `tenant`, `roles`, `scope`, `exp`. Refresh tokens stored in Redis.
- `api/deps.py`: `current_user()` decodes JWT, loads user, sets tenant context; `require(*perms)` dependency factory enforces RBAC and returns RFC 7807 `403`.
- Passwords hashed with `argon2`.

**Testing**:
- `Unit: valid credentials → JWT with correct claims; expired token → 401`.
- `Integration: client_credentials grant → machine token with scopes`.
- `Integration: learner calls admin endpoint → 403 problem+json`.
- `Unit: argon2 verify rejects tampered hash`.

---

## Phase 2: Skills Taxonomy & Open Dataset Import

### Purpose
Build the skills intelligence backbone: frameworks, categories, skills, skill relationships, and job roles — grounded in the open ESCO and O*NET datasets per `standards.md`. This is the differentiating "open skills record" foundation; gap analysis and pathways (Phase 4) depend entirely on it.

### Tasks

#### 2.1 — Skills taxonomy models

**What**: Implement `skill_frameworks`, `skill_categories` (tree), `skills`, `skill_relationships`, `job_roles`, `job_role_skills` from Data Model Suggestion 3's skills domain, aligned to IEEE 1484.20.1 competency-definition semantics.

**Design**:
- `skills.external_uri` holds the ESCO URI or O*NET element id; `skills.labels` JSONB stores multilingual labels (`{"en": "...", "fr": "..."}`) — the hybrid model's flexibility for ESCO's multilingual data.
- `proficiency_scale` enum (`binary|three_level|five_level|ten_level|percentage`), normalised internally to 0–10.
- GIN full-text index on skill name + JSONB labels; `pg_trgm` index for fuzzy skill search.
- `skill_relationships.relationship_type` ∈ `prerequisite|related|supersedes|component_of|alternative` with `strength` 0–1.

**Testing**:
- `Unit: proficiency normaliser maps each scale → 0–10 correctly`.
- `Integration: insert skill with multilingual JSONB labels; query by lang`.
- `Integration: self-referential prerequisite relationship; CHECK rejects self-loop`.
- `Integration: trigram fuzzy search "kubernets" matches "Kubernetes"`.

#### 2.2 — ESCO importer

**What**: Import the ESCO skills/occupations taxonomy from the official RDF/CSV download into a framework.

**Design**:
- `integrations/taxonomy/esco.py`: parse ESCO SKOS (rdflib) — `concepts` → skills, `broader`/`narrower` → categories, `isEssentialSkillFor`/`isOptionalSkillFor` → `job_role_skills` (importance critical/preferred). Idempotent upsert keyed on `external_uri`.
- Run as a Celery task `import_esco_framework(tenant_id, dataset_path)` with progress recorded in `integration_sync_log`.
- Ships a small committed ESCO subset fixture (~200 skills) for tests.

**Testing**:
- `Fixture: import 200-skill ESCO subset → frameworks/categories/skills counts match expected`.
- `Unit: SKOS broader/narrower → category tree depth correct`.
- `Integration: re-running import is idempotent (no duplicates)`.
- `Integration: malformed RDF → task fails, sync_log status='failed' with error detail`.

#### 2.3 — O*NET importer

**What**: Import O*NET occupations + skills via the O*NET Web Services API (API-key auth) for North American coverage.

**Design**:
- `integrations/taxonomy/onet.py`: httpx client; map O*NET-SOC codes → `job_roles.onet_code`, element ids → `skills.external_uri`, importance/level scores → `job_role_skills.required_level`/`importance`.
- Respect O*NET rate limits with backoff; cache responses.

**Testing**:
- `Integration (mocked O*NET API): occupation payload → job_role + mapped skills`.
- `Unit: O*NET importance score → importance enum mapping`.
- `Integration (mocked): 429 → retried with backoff`.

#### 2.4 — Skills & job-role CRUD API

**What**: REST endpoints to manage frameworks, skills, relationships, job roles, and role-skill maps.

**Design** (OpenAPI 3.1, RFC 8288 `Link` pagination):
- `GET /api/v1/skills?framework_id=&q=&type=&page=&size=` → paginated, full-text/trigram search.
- `POST /api/v1/skills`, `PATCH /api/v1/skills/{id}`, `DELETE` (soft via `is_active`).
- `POST /api/v1/skills/{id}/relationships`.
- `GET/POST /api/v1/job-roles`, `PUT /api/v1/job-roles/{id}/skills` (bulk set required levels).
- All write endpoints require `skills:write`; emit `audit_log` rows.

**Testing**:
- `Integration: paginated search returns Link header with next/prev`.
- `Integration: create skill missing required field → 422 with field path`.
- `Integration: bulk set role skills replaces prior mapping atomically`.
- `Integration: delete skill referenced by job_role → soft-deletes, mapping preserved`.

---

## Phase 3: LMS Core — Content, SCORM/xAPI Runtime & Enrolments

### Purpose
Deliver the table-stakes LMS: course catalogue, content asset hosting (SCORM 1.2/2004, cmi5), the SCORM runtime API, the xAPI Learning Record Store, enrolments, and progress tracking. This is the operational heart of the product and where most competitor parity lives.

### Tasks

#### 3.1 — Content catalogue models & upload

**What**: `content_providers`, `courses`, `course_modules` (tree), `content_assets`, `course_skills`, `assessments`, `assessment_questions`, `question_options` per Data Model Suggestion 3's content domain, with package upload to object storage.

**Design**:
- `courses.metadata` JSONB carries content-type-specific data (SCORM manifest summary, LTI client/deployment ids, AI-gen prompt/model/version) — the hybrid model's core benefit.
- `services/content_service.py`: accept multipart SCORM/cmi5 zip → store in S3/MinIO (`content_assets.storage_url`, `checksum_sha256`) → parse `imsmanifest.xml` (SCORM) or `cmi5.xml` to extract SCO identifiers + entry point into `metadata`.
- `course_skills` links courses to taught skills + `skill_level_taught` (drives gap-closing pathways).
- IEEE 1484.12.1 LOM fields surfaced in course metadata for discovery.

**Testing**:
- `Fixture: upload sample SCORM 1.2 zip → asset stored, manifest parsed, entry point captured`.
- `Fixture: upload cmi5 package → AU list extracted`.
- `Integration: checksum mismatch on re-upload → 409`.
- `Unit: course_modules tree ordering by sort_order within parent`.

#### 3.2 — SCORM runtime API + player bridge

**What**: SCORM 1.2/2004 RTE: `LMSInitialize/Initialize`, `GetValue`, `SetValue`, `Commit`, `Finish` semantics backed by `scorm_runtime_data`.

**Design**:
- `services/scorm_runtime.py` + `api/v1/scorm.py`: a JSON RTE bridge the client-side SCORM API wrapper calls.
  - `POST /scorm/{enrolment_id}/initialize` → returns persisted CMI data model (or fresh defaults) for the attempt; live session held in Redis.
  - `POST /scorm/{enrolment_id}/set` `{element, value}` → validates against SCORM data model element rules.
  - `POST /scorm/{enrolment_id}/commit` → persists Redis session to `scorm_runtime_data`; maps `cmi.core.lesson_status`/`cmi.completion_status`+`cmi.success_status` → `enrolments.status`/`completion_pct`/`score`.
- On `passed`/`completed`, emit a `learning_event` and an xAPI statement (3.3).
- `suspend_data` (up to 64k for SCORM 2004) stored in Redis live, persisted on commit.

**Testing**:
- `Unit: lesson_status "passed" → enrolment completed + score mapped`.
- `Unit: SetValue on read-only element (cmi.core.score.max in 1.2 write rules) → error code 404`.
- `Integration: initialize→set→commit→finish round-trip persists runtime row; suspend_data survives re-initialize`.
- `E2E (Playwright): launch sample SCORM, complete it, enrolment shows completed`.

#### 3.3 — xAPI Learning Record Store

**What**: A conformant xAPI 1.0.3 LRS Statement API backed by `xapi_statements`.

**Design** (per `standards.md` xAPI spec):
- `PUT /api/v1/xapi/statements?statementId=` (client-set UUID), `POST /api/v1/xapi/statements`, `GET /api/v1/xapi/statements?agent=&verb=&activity=&since=&until=&limit=` with `X-Experience-API-Version: 1.0.3` header enforcement.
- Statement stored both as a validated JSONB document (`statement_json`) **and** with hot columns extracted (actor, verb_id, object_id, result_score_scaled, timestamp) for indexed querying — the hybrid pattern.
- Voiding via `http://adlnet.gov/expapi/verbs/voided` sets `voided=true` on the referenced statement.
- Auth: OAuth 2.0 client-credentials scope `xapi:write`/`xapi:read`.

**Testing**:
- `Integration: PUT statement with conflicting id+different content → 409`.
- `Integration: GET filtered by verb+agent → only matching, paginated`.
- `Unit: statement missing actor/verb/object → 400 (xAPI required props)`.
- `Integration: voiding statement marks target voided, excluded from default GET`.

#### 3.4 — Enrolments, modules progress & certifications

**What**: `enrolments`, `module_progress`, `certifications`, `user_certifications` with enrolment lifecycle.

**Design**:
- State machine: `enrolled → in_progress → completed | failed | expired | withdrawn`. Transitions in `services/enrolment_service.py`; illegal transitions raise domain error.
- `enrolment_type` ∈ `voluntary|assigned|compliance_required|ai_recommended`.
- On completion of a course with linked `certifications.validity_days`, issue `user_certifications` with `expires_at`; generate a PDF certificate (reportlab) to S3.
- `POST /api/v1/enrolments`, `GET /api/v1/users/{id}/enrolments`, `POST /api/v1/enrolments/{id}/withdraw`.

**Testing**:
- `Unit: state machine rejects completed→in_progress`.
- `Integration: completing cert-bearing course issues user_certification with correct expiry`.
- `Integration: duplicate active enrolment (same user+course) prevented`.
- `Integration: certificate PDF generated and retrievable via signed URL`.

---

## Phase 4: AI Skill Gap Analysis & Pathway Generation

### Purpose
Deliver the core AI-native value: compute individual and team skill gaps against job-role targets, then generate optimised learning pathways that close those gaps using the skill prerequisite graph and course-skill mappings. This is the heart of the product and the primary differentiator over plain LMSs.

### Tasks

#### 4.1 — LLM client gateway

**What**: A provider-agnostic LLM wrapper with structured output, retries, cost logging, and versioned prompts.

**Design**:
- `ai/client.py`: `async def complete(prompt: str, *, schema: type[BaseModel] | None, model: str) -> BaseModel | str` over LiteLLM; JSON-mode/structured-output enforced via the Pydantic `schema`; exponential backoff on rate limits; every call logged (model, tokens, cost, latency, prompt-version) to a `llm_call_log` table.
- `ai/prompts/` Jinja2 templates, each with a `# version: N` header so outputs are reproducible/auditable (ISO/IEC 19796 quality traceability).

**Testing**:
- `Unit (mocked LiteLLM): structured call returns parsed Pydantic model`.
- `Unit: invalid JSON from model → one repair retry, then error`.
- `Unit: rate-limit error → backoff then success; cost logged`.

#### 4.2 — Skill gap analysis engine

**What**: Compute `skill_gap_analyses` + `skill_gap_details` for a user or team against a target job role.

**Design**:
- `services/gap_analysis.py`:
  ```python
  def analyze(target_type, target_id, job_role_id) -> SkillGapAnalysis:
      required = role_skills(job_role_id)            # skill_id -> required_level, importance
      current  = effective_user_skills(target_id)    # best of self/manager/assessment/ai_inferred
      for each required skill: gap = max(0, required - current)
      priority = f(importance, gap_size)             # critical+large gap => critical
      overall_readiness_pct = weighted by importance
  ```
- Team analysis aggregates members (min/avg/coverage per skill).
- `gap_size` is the DB-generated stored column from the data model.

**Testing**:
- `Unit: required 4, current 1, importance critical → gap 3, priority critical`.
- `Unit: readiness weighting — all critical met, preferred unmet → high but <100%`.
- `Integration: team analysis aggregates 3 members, identifies single-point-of-failure skill (coverage=1)`.

#### 4.3 — AI learning pathway generation

**What**: Generate a `learning_pathways` + ordered `pathway_steps` that close a gap analysis, respecting skill prerequisites.

**Design**:
- `services/pathway_service.py` + `ai/pathway_gen.py`:
  1. From gap details, collect target skills (priority-ordered).
  2. Expand via `skill_relationships` (recursive CTE) to include `prerequisite` skills → topologically order (the Suggestion-4 graph problem solved in-process via CTE + toposort).
  3. For each skill, candidate courses from `course_skills` ranked by content `quality_score` + `skill_level_taught` coverage.
  4. LLM (`pathway_gen` prompt) sequences courses into a coherent pathway with rationale, dedupes overlap, and estimates duration; output validated against a `PathwayPlan` Pydantic schema.
  5. Persist pathway (`pathway_type='ai_generated'`) + steps with `prerequisite_step_id` links.
- Prompt template includes learner context, gap list, candidate courses (id/title/skills/quality), and target role.

**Testing**:
- `Unit: prerequisite expansion + toposort orders foundational skills first`.
- `Integration (mocked LLM): gap → pathway with steps covering all critical-gap skills`.
- `Unit: course ranking prefers higher quality_score at equal coverage`.
- `Integration: generated pathway persisted with valid prerequisite_step ordering`.

#### 4.4 — Gap & pathway API + dashboards data

**What**: Endpoints powering individual/team gap dashboards and pathway management.

**Design**:
- `POST /api/v1/gap-analyses` `{target_type,target_id,job_role_id}` → analysis + details.
- `GET /api/v1/gap-analyses/{id}`; `GET /api/v1/teams/{org_id}/skill-heatmap` (backed by `mv_team_skill_heatmap`).
- `POST /api/v1/pathways/generate` `{user_id, job_role_id}` → enqueues Celery job, returns `202` + job id; `GET /api/v1/pathways/{id}` polls result.
- `POST /api/v1/pathways/{id}/enroll` enrols the user into all pathway courses.

**Testing**:
- `Integration: heatmap endpoint returns avg/min/coverage per skill for a team`.
- `Integration: generate is async → 202 + pollable job; completes to pathway`.
- `Integration: enroll-from-pathway creates enrolments for each course step`.

---

## Phase 5: HRIS Provisioning (SCIM 2.0) & Enterprise SSO

### Purpose
Make the platform deployable into real enterprises: automated user lifecycle via SCIM 2.0 from Workday/BambooHR, and SSO via SAML 2.0 / OpenID Connect. Without these, no enterprise can adopt it; they are listed table-stakes in `features.md`.

### Tasks

#### 5.1 — SCIM 2.0 server

**What**: RFC 7643/7644-compliant SCIM endpoints for inbound user/group provisioning.

**Design**:
- `integrations/scim/`: `/scim/v2/Users` (GET list with filter, GET/{id}, POST, PUT, PATCH, DELETE), `/scim/v2/Groups`, `/scim/v2/ServiceProviderConfig`, `/scim/v2/Schemas`.
- Map SCIM `userName`/`externalId`/`emails`/`name`/`active`/`manager` → `users` columns; unmapped SCIM attrs → `users.custom_fields` JSONB (hybrid model).
- Filter parsing via `scim2-filter-parser` → SQLAlchemy. Auth via OAuth client-credentials (`scim` scope). `active=false` → user `status='inactive'` (offboarding), preserving learning history (GDPR-aware soft state).

**Testing**:
- `Integration: POST /Users → user created, externalId stored, 201 with Location`.
- `Integration: PATCH active=false → status inactive, enrolments retained`.
- `Integration: GET /Users?filter=userName eq "x" → matched user`.
- `Integration: GET /ServiceProviderConfig → advertises supported features`.

#### 5.2 — HRIS connectors (Workday, BambooHR)

**What**: Scheduled pull-sync connectors as an alternative/complement to SCIM push.

**Design**:
- `integrations/hris/base.py` defines `HRISConnector` (`fetch_users()`, `fetch_org_units()`); `workday.py` (OAuth ISU creds, REST), `bamboohr.py` (API key).
- Celery beat task `sync_hris(tenant_id, connector)` upserts users/orgs, records counts in `integration_sync_log`, reconciles departures (mark inactive).

**Testing**:
- `Integration (mocked Workday): fetch → upsert creates/updates users, org tree built`.
- `Integration (mocked BambooHR): departed employee → marked inactive`.
- `Unit: sync_log captures created/updated/failed counts`.

#### 5.3 — SAML 2.0 & OIDC SSO

**What**: Enterprise login via SAML and OIDC, linking to `identity_providers`/`user_identity_links`.

**Design**:
- `integrations/sso/saml.py` (python3-saml): `GET /auth/saml/{idp_id}/login` (AuthnRequest), `POST /auth/saml/{idp_id}/acs` (validate assertion, match `NameID` → `user_identity_links`, JIT-provision if configured, issue platform JWT).
- `integrations/sso/oidc.py` (Authlib): authorization-code flow against tenant IdP (`issuer_url`, `client_id`); map `sub` → link.
- Encrypted client secrets via `security/crypto.py` (Fernet, key from env).

**Testing**:
- `Integration (mocked IdP): valid SAML assertion → linked user, platform JWT issued`.
- `Integration: unknown NameID with JIT off → 403; JIT on → user provisioned`.
- `Unit: client_secret encrypted at rest, decrypted for flow`.
- `Integration: OIDC code exchange → user matched by sub claim`.

---

## Phase 6: AI Authoring, Manager Coaching & Notifications

### Purpose
Ship the v1.1 AI differentiators and engagement layer: generate course structure + assessment questions from a topic/document, AI manager coaching suggestions, content quality scoring, and Slack/Teams learning nudges. These move the platform from passive LMS to active, AI-driven L&D.

### Tasks

#### 6.1 — AI course authoring

**What**: Generate a course outline (modules) + assessment questions from a topic or uploaded document.

**Design**:
- `ai/authoring.py`: input `{topic | document_text, target_skills, difficulty}` → LLM (`authoring` prompt) → `GeneratedCourse` schema (modules with learning objectives, draft content, assessment questions + options + correct answers + explanations). Persisted as a `draft` course with `content_type='ai_generated'`, `metadata` recording model + prompt version (ISO/IEC 40180 quality traceability).
- Long documents chunked; map-reduce summarisation before outline generation.

**Testing**:
- `Integration (mocked LLM): topic → course with N modules + questions, status=draft`.
- `Unit: each generated MCQ has exactly one correct option`.
- `Integration: document upload → chunked → coherent outline`.

#### 6.2 — Compliance question generation

**What**: Generate assessment questions directly from a policy document (ISO/IEC 23988 assessment practice).

**Design**:
- `ai/compliance_qgen.py`: policy doc → questions tagged with `source_document_url` and `ai_generated=true`, linked to a compliance course/assessment. Includes a "grounding" check: each question must cite a span from the source doc.

**Testing**:
- `Integration (mocked LLM): policy text → questions each carrying source citation`.
- `Unit: question without grounding citation rejected`.

#### 6.3 — Content quality scoring

**What**: Compute `courses.quality_score` from freshness, learner ratings, and source credibility (a `features.md` differentiator + ISO/IEC 19796 alignment).

**Design**:
- `services/content_quality.py`: `score = w_fresh*freshness(freshness_date) + w_rating*avg_rating + w_cred*source_credibility`. Weights configurable per tenant in `settings_json`. Recomputed nightly (Celery beat) and on rating events.
- Stale content (freshness beyond threshold) flagged for review and surfaced to admins.

**Testing**:
- `Unit: fresh + highly-rated + credible → high score`.
- `Unit: content past freshness threshold → flagged stale`.
- `Integration: new learner rating triggers recompute`.

#### 6.4 — Manager coaching assistant

**What**: For a team member's gap, generate actionable coaching suggestions for the manager.

**Design**:
- `ai/coaching.py`: input = member gap analysis + role context → LLM (`coaching` prompt) → ranked, concrete coaching actions (recommend pathway, stretch assignment, pairing, check-in cadence) with rationale. Surfaced in the manager dashboard, not auto-applied.
- `GET /api/v1/users/{id}/coaching-suggestions` (requires `manager` role + management relationship).

**Testing**:
- `Integration (mocked LLM): gap → ≥3 ranked suggestions with rationale`.
- `Integration: manager of non-report → 403`.

#### 6.5 — Slack & Teams nudge notifications

**What**: Deliver learning nudges (due training, recommended pathway, streaks) to Slack/Teams plus in-app/email.

**Design**:
- `db/models/integration.py` `notifications` table; `integrations/messaging/{slack,teams}.py` adapters implementing `send(user, message)`.
- Celery task `dispatch_notifications` consumes a queue; nudge triggers from `learning_events` (overdue compliance, idle learner, pathway available). Records `nudge_sent`/`nudge_clicked` events for ROI engagement metrics.

**Testing**:
- `Integration (mocked Slack API): overdue compliance → Slack message + notification row`.
- `Unit: nudge dedup — same nudge not sent twice within cooldown`.
- `Integration: nudge click webhook → nudge_clicked learning_event`.

---

## Phase 7: Compliance Automation & ROI Analytics

### Purpose
Close the enterprise-governance and the unsolved-ROI gaps from `research.md`: automated compliance assignment with audit-trail export, and ROI reporting that correlates learning with business KPIs — the area `features.md` calls out as unsolved across all incumbents.

### Tasks

#### 7.1 — Compliance automation engine

**What**: `compliance_rules` evaluation that auto-assigns courses by target (all/org/role/group) on a recurring frequency with grace periods, plus audit export.

**Design**:
- `services/compliance_service.py`: Celery beat sweep evaluates active rules → resolves target population → creates `enrolment_type='compliance_required'` enrolments with `due_date = now + grace_period_days`; re-assigns when `frequency` window lapses (annual/quarterly/etc.).
- Compliance status from `vw_compliance_status` view (data model): `not_enrolled|compliant|in_progress|overdue`.
- `GET /api/v1/compliance/status?org_id=`; `GET /api/v1/compliance/audit-export?from=&to=&format=csv|json` produces an immutable, signed audit trail (GDPR Art. 39 record-keeping).

**Testing**:
- `Integration: rule targeting a job role assigns all role-holders, due_date set`.
- `Unit: annual frequency re-assigns after window; not before`.
- `Integration: audit export rows match enrolment + completion history exactly`.
- `Integration: overdue computation matches due_date < now & not completed`.

#### 7.2 — Business KPI ingestion

**What**: `business_kpi_definitions` + `business_kpi_values` import for ROI correlation.

**Design**:
- `POST /api/v1/kpis` (define), `POST /api/v1/kpis/{id}/values` (bulk CSV/JSON upload of per-user/per-org period values). `kpi_category` ∈ sales|retention|promotion|quality|… ; `higher_is_better` controls delta interpretation.

**Testing**:
- `Integration: CSV upload of KPI values → rows per user/period`.
- `Unit: malformed period rejected with row-level error report`.

#### 7.3 — ROI attribution & reporting

**What**: Correlate learning interventions with KPI changes and produce `roi_reports` + `roi_report_correlations`.

**Design**:
- `services/roi_service.py`: for a cohort that completed a course/pathway, compute pre/post KPI deltas vs a control cohort (matched non-learners); statistical significance via `scipy`/`statsmodels` (t-test / pre-post with control). `ai/roi_attribution.py` produces a narrative summary and flags weak/insignificant correlations.
- **Feedback loop** (`features.md` differentiator): auto-flag pathways whose targeted KPI did not improve (`delta_pct ≤ 0` or not significant) for content iteration.
- Reports generated async (Celery), `status: generating → generated`.

**Testing**:
- `Unit: pre/post with positive significant delta → is_significant true, delta_pct correct`.
- `Unit: no control cohort → falls back to pre/post, attribution_method recorded`.
- `Integration: pathway with no KPI improvement → flagged in feedback loop`.
- `Integration (mocked LLM): correlation set → narrative summary generated`.

---

## Phase 8: Content Provider Connectors & Web Portals

### Purpose
Connect external content libraries (LinkedIn Learning, Coursera) and ship the three role-based web portals so the platform is usable by learners, managers, and admins — not just via API.

### Tasks

#### 8.1 — Content provider connectors

**What**: Ingest catalogue + completion data from LinkedIn Learning and Coursera into `courses`/`enrolments`.

**Design**:
- `integrations/content/base.py` `ContentConnector` (`sync_catalog()`, `fetch_completions()`); `linkedin_learning.py` (OAuth 2.0, LinkedIn Partner API), `coursera.py`. External courses created with `provider_id` + `external_course_id`, completions mapped to enrolments and emitted as xAPI statements.
- Celery beat catalog sync; completion webhooks where available.

**Testing**:
- `Integration (mocked LinkedIn API): catalog sync → external courses created/updated`.
- `Integration (mocked): completion → enrolment completed + xAPI statement`.
- `Unit: external_course_id dedup prevents duplicate course rows`.

#### 8.2 — Web app shell, auth & learner portal

**What**: Next.js app with SSO/OAuth login, role-aware routing, and the learner experience.

**Design**:
- `web/`: shadcn/ui + TanStack Query. Auth via platform OAuth/OIDC, JWT in httpOnly cookie.
- Learner portal: personalised feed (recommended pathways, due compliance), skill profile (current vs target levels), course player (SCORM iframe + RTE bridge from 3.2), certificates wallet.

**Testing**:
- `E2E (Playwright): login → learner home shows assigned + recommended learning`.
- `E2E: open SCORM course, complete, certificate appears in wallet`.
- `Component: skill profile renders current/target bars from API`.

#### 8.3 — Manager & admin portals

**What**: Manager team dashboard (gap heatmap, coaching suggestions) and admin console (taxonomy, courses, compliance rules, integrations, reports).

**Design**:
- Manager: team skill-gap heatmap (from 4.4), per-report coaching suggestions (6.4), team compliance status.
- Admin: framework import triggers, course catalogue + AI authoring (6.1), compliance rule builder (7.1), SCIM/HRIS/SSO config, ROI report viewer (7.3), content quality/stale flags (6.3).

**Testing**:
- `E2E: manager views heatmap, opens a member's coaching suggestions`.
- `E2E: admin creates compliance rule → learners auto-assigned`.
- `E2E: admin triggers ESCO import → skills appear in catalogue`.

---

## Phase 9: MCP Server, Credential Export & Hardening

### Purpose
Ship the high-priority AI-era differentiator (MCP server exposing the platform inside enterprise LLMs), portable verifiable credentials, and the security/compliance hardening required for enterprise procurement (OWASP, GDPR, observability).

### Tasks

#### 9.1 — MCP server

**What**: A Model Context Protocol server exposing catalogue search, learner progress, certification status, and skill gaps to LLM tools (Claude, Copilot, ChatGPT) — matching Docebo's differentiator.

**Design**:
- `mcp/` using the `mcp` SDK; tools: `search_courses(query, skill)`, `get_learner_progress(user)`, `get_certifications(user)`, `get_skill_gaps(user, role)`, `recommend_pathway(user, role)`. OAuth-scoped, tenant-isolated; read-only by default.

**Testing**:
- `Integration: MCP search_courses tool returns catalogue matches`.
- `Integration: get_learner_progress enforces tenant + scope`.
- `Unit: tool schemas validate against MCP spec`.

#### 9.2 — Verifiable credential / Open Badges export

**What**: Export a learner's certifications as Open Badges 3.0 / W3C Verifiable Credentials 2.0 for cross-platform portability (the unsolved gap in `features.md`).

**Design**:
- `services/credential_service.py`: build a VC 2.0 JSON-LD credential per `user_certification`, signed (Ed25519); `GET /api/v1/users/{id}/credentials/{cert_id}/export?format=ob3|vc`. Skill record exportable as JSON-LD "skills wallet".

**Testing**:
- `Unit: exported credential validates against Open Badges 3.0 context`.
- `Integration: signature verifies with published public key`.

#### 9.3 — Security, GDPR & observability hardening

**What**: OWASP Top 10 mitigations, GDPR right-to-erasure with analytics-safe anonymisation, audit completeness, and observability.

**Design**:
- `security/`: rate limiting (Redis), input validation, RLS verification tests, secret encryption, security headers; audited against OWASP Top 10 (2021) — esp. A01 (RLS/RBAC), A03 (parameterised queries), A07 (auth).
- GDPR: `DELETE /api/v1/users/{id}/personal-data` anonymises PII while preserving aggregate analytics integrity (replace identifiers, retain de-identified `learning_events`); erasure recorded in `audit_log`.
- Observability: structured JSON logging, Prometheus metrics, OpenTelemetry tracing across API → worker → LLM calls.

**Testing**:
- `Integration: cross-tenant access attempt blocked (RLS) at API and DB layers`.
- `Integration: erasure anonymises user, aggregate completion counts unchanged`.
- `Integration: rate limit returns 429 after threshold`.
- `Security: automated OWASP ZAP baseline scan passes in CI`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Multi-Tenancy & Identity        ── required by everything
    │
    ├── Phase 2: Skills Taxonomy & Dataset Import     ── requires P1
    │       │
    │       └── Phase 4: AI Gap Analysis & Pathways   ── requires P2, P3
    │
    ├── Phase 3: LMS Core (content/SCORM/xAPI/enrol)  ── requires P1
    │       │
    │       └── Phase 4 (also requires P3)
    │
    ├── Phase 5: HRIS (SCIM) & SSO                    ── requires P1   (parallel with P2/P3)
    │
    ├── Phase 6: AI Authoring, Coaching, Notifications── requires P3, P4
    │
    ├── Phase 7: Compliance & ROI Analytics           ── requires P3 (compliance), P4 (ROI cohorts)
    │
    ├── Phase 8: Content Connectors & Web Portals     ── requires P3, P4, P6, P7
    │
    └── Phase 9: MCP, Credentials & Hardening         ── requires P3, P4 (MCP/creds); P1–P8 (hardening)
```

**Parallelism opportunities:**
- After Phase 1: **Phase 2, Phase 3, and Phase 5** can be developed concurrently (independent domains sharing only the identity/DB foundation).
- After Phases 3 & 4: **Phase 6 and Phase 7** can proceed concurrently (distinct service areas).
- Phase 8 (portals) and Phase 9 (MCP/credentials) can begin in parallel once their backend dependencies (P3/P4/P6/P7) land; hardening sub-tasks in 9.3 are continuous but gated for final sign-off.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks implemented to the design specified.
2. All unit and integration tests pass; new code has meaningful coverage (≥80% on touched modules).
3. `ruff` lint + format pass with no errors.
4. `mypy --strict` passes for all new/changed Python modules.
5. Alembic migration(s) created, apply cleanly forward, and revert cleanly down.
6. Docker image builds; `docker-compose up` brings the stack healthy.
7. The phase's feature works end-to-end (E2E/Playwright test where a UI or full workflow exists).
8. New API endpoints appear correctly in the auto-generated OpenAPI 3.1 spec.
9. New config options documented in `.env.example` and README.
10. RLS/tenant isolation verified for any new tenant-scoped tables.
11. New mutating operations write `audit_log` entries.
12. Relevant standards conformance asserted by test where applicable (xAPI statement shape, SCIM ServiceProviderConfig, SCORM RTE error codes, Open Badges context).
```
