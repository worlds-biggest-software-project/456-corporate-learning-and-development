# Data Model Suggestion 3: Hybrid Relational + Document (PostgreSQL JSONB)

## Overview

This model uses PostgreSQL as a single database engine but strategically mixes fully normalized relational tables for stable, well-understood data with JSONB columns for data that is inherently variable, extensible, or schema-fluid. The goal is to get the best of both worlds: referential integrity and efficient JOINs where structure is known, and document-style flexibility where the shape of data varies by tenant, content type, or integration source.

For a corporate L&D platform, the hybrid approach is particularly well-suited because:

- **xAPI statements and SCORM runtime data have fixed envelopes but variable content**: The Actor-Verb-Object structure is stable, but extensions, context activities, and interaction data vary wildly between content providers and authoring tools.
- **Skills taxonomies are structurally diverse**: ESCO uses URIs and multi-language labels, O*NET uses numeric codes and detailed element breakdowns, and custom tenant taxonomies may have entirely different hierarchies. A single relational schema cannot anticipate all variants.
- **Course metadata varies by content type**: A SCORM package has manifest data, launch URLs, and SCO identifiers. An LTI tool has client IDs, deployment IDs, and custom parameters. An AI-generated course has prompt templates, model versions, and generation parameters. JSONB lets each content type carry its own metadata without cluttering a shared table with nullable columns.
- **Tenant customization is essential**: Enterprise customers expect custom fields on users, courses, and enrolments. JSONB columns handle this without DDL migrations.
- **Assessment question types are open-ended**: Multiple choice, drag-and-drop, simulations, and AI-graded responses all have different data shapes.

---

## Technology Recommendations

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Database | PostgreSQL 16+ | Native JSONB with GIN/GPath indexing, CTEs, window functions, full-text search |
| JSONB Validation | pg_jsonschema or application-level | Validate JSONB content against JSON Schema to maintain data quality |
| Connection Pool | PgBouncer | Transaction-level pooling |
| ORM | Prisma, Drizzle, or SQLAlchemy | All support JSONB column mapping natively |
| Search | PostgreSQL GIN indexes + optional OpenSearch | GIN indexes handle most search; add OpenSearch only for complex faceted search at scale |
| Migrations | Flyway or Prisma Migrate | Schema migrations for relational parts; JSONB evolves without DDL |
| Caching | Redis | Cache frequently-accessed JSONB documents (skill profiles, course metadata) |

---

## Design Principles

1. **Relational columns for query predicates**: Any column you filter, sort, or JOIN on gets a proper typed column with an index.
2. **JSONB for variable/extensible payloads**: Data that varies by type, tenant, or integration goes in JSONB.
3. **Extract on read, not on write**: When a JSONB field becomes frequently queried, extract it to a relational column with a backfill migration. The JSONB copy remains as the extensible store.
4. **GIN indexes on JSONB**: Create GIN indexes on JSONB columns that need querying. Use `jsonb_path_ops` for containment queries.
5. **JSON Schema validation**: Define and enforce JSON Schema contracts at the application layer for critical JSONB columns to prevent garbage data.

---

## Schema Definition

### Domain 1: Identity & Organization

```sql
-- =============================================================
-- TENANTS
-- =============================================================
CREATE TABLE tenants (
    tenant_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
    -- JSONB for tenant-specific settings: branding, feature flags, custom fields config
    settings        JSONB NOT NULL DEFAULT '{
        "branding": {"primary_color": "#1a73e8", "logo_url": null},
        "features": {"ai_pathways": true, "compliance_module": true, "roi_reporting": false},
        "custom_fields": {
            "user": [],
            "course": [],
            "enrolment": []
        },
        "integrations": {
            "slack_webhook_url": null,
            "teams_webhook_url": null
        },
        "compliance": {
            "default_grace_period_days": 30,
            "escalation_levels": 3
        }
    }',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =============================================================
-- ORGANIZATIONS
-- =============================================================
CREATE TABLE organizations (
    org_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    parent_org_id   UUID REFERENCES organizations(org_id),
    name            VARCHAR(255) NOT NULL,
    org_type        VARCHAR(50) NOT NULL,
    code            VARCHAR(50),
    -- JSONB for org-specific metadata: cost center, budget, location info
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Materialized path for efficient hierarchy queries
    path            TEXT NOT NULL DEFAULT '/',       -- e.g., '/root-id/div-id/dept-id/'
    depth           INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_organizations_tenant ON organizations (tenant_id);
CREATE INDEX idx_organizations_path ON organizations USING gist (path gist_trgm_ops);

-- =============================================================
-- USERS
-- =============================================================
CREATE TABLE users (
    user_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    org_id          UUID REFERENCES organizations(org_id),
    -- Stable, frequently queried fields as relational columns
    external_id     VARCHAR(255),
    username        VARCHAR(255) NOT NULL,
    email           VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    display_name    VARCHAR(255),
    job_title       VARCHAR(255),
    manager_id      UUID REFERENCES users(user_id),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    last_login_at   TIMESTAMPTZ,
    -- JSONB for variable user profile data
    -- Includes: SCIM enterprise extension fields, custom fields, preferences, HRIS sync data
    profile         JSONB NOT NULL DEFAULT '{
        "employee_number": null,
        "hire_date": null,
        "cost_center": null,
        "division": null,
        "timezone": "UTC",
        "locale": "en",
        "avatar_url": null,
        "phone_numbers": [],
        "addresses": [],
        "scim_metadata": {},
        "custom_fields": {},
        "notification_preferences": {
            "email": true,
            "slack": false,
            "teams": false,
            "push": true,
            "learning_nudges": true,
            "compliance_reminders": true
        }
    }',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, username),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_users_tenant_status ON users (tenant_id, status);
CREATE INDEX idx_users_org ON users (org_id);
CREATE INDEX idx_users_manager ON users (manager_id);
CREATE INDEX idx_users_external_id ON users (tenant_id, external_id);
-- GIN index for searching within profile JSONB
CREATE INDEX idx_users_profile ON users USING gin (profile jsonb_path_ops);
```

### Domain 2: Skills Taxonomy (Heavily JSONB)

```sql
-- =============================================================
-- SKILL FRAMEWORKS
-- =============================================================
CREATE TABLE skill_frameworks (
    framework_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            VARCHAR(255) NOT NULL,
    source          VARCHAR(50) NOT NULL,            -- 'esco', 'onet', 'custom', 'imported'
    version         VARCHAR(50),
    -- JSONB for source-specific metadata
    -- ESCO: {"uri_prefix": "http://data.europa.eu/esco/skill/...", "language": "en", "release": "v1.1.1"}
    -- O*NET: {"version": "28.1", "element_types": ["abilities", "skills", "knowledge"]}
    -- Custom: {"proficiency_definitions": [...], "approval_workflow": true}
    source_metadata JSONB NOT NULL DEFAULT '{}',
    is_default      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =============================================================
-- SKILLS (hybrid: relational core + JSONB extensions)
-- =============================================================
CREATE TABLE skills (
    skill_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework_id    UUID NOT NULL REFERENCES skill_frameworks(framework_id) ON DELETE CASCADE,
    -- Relational: queryable, joinable fields
    parent_skill_id UUID REFERENCES skills(skill_id),   -- hierarchical taxonomy
    name            VARCHAR(500) NOT NULL,
    skill_type      VARCHAR(30) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- Materialized path for hierarchy
    path            TEXT NOT NULL DEFAULT '/',
    depth           INTEGER NOT NULL DEFAULT 0,
    -- JSONB for everything that varies by framework source
    details         JSONB NOT NULL DEFAULT '{
        "description": null,
        "external_uri": null,
        "alt_labels": [],
        "proficiency_scale": {
            "type": "five_level",
            "levels": [
                {"value": 1, "label": "Awareness", "description": "Basic understanding of concepts"},
                {"value": 2, "label": "Novice", "description": "Can perform with guidance"},
                {"value": 3, "label": "Intermediate", "description": "Can perform independently"},
                {"value": 4, "label": "Advanced", "description": "Can lead and mentor others"},
                {"value": 5, "label": "Expert", "description": "Recognized authority"}
            ]
        },
        "esco": {
            "uri": null,
            "concept_type": null,
            "broader_concepts": [],
            "narrower_concepts": [],
            "related_occupations": []
        },
        "onet": {
            "element_id": null,
            "element_name": null,
            "scale_id": null,
            "category": null
        },
        "tags": [],
        "keywords": [],
        "related_certifications": []
    }',
    -- Full-text search vector (auto-maintained trigger)
    search_vector   TSVECTOR GENERATED ALWAYS AS (
        to_tsvector('english', name || ' ' || COALESCE(details->>'description', ''))
    ) STORED,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_skills_framework ON skills (framework_id);
CREATE INDEX idx_skills_parent ON skills (parent_skill_id);
CREATE INDEX idx_skills_type ON skills (skill_type);
CREATE INDEX idx_skills_path ON skills (path);
CREATE INDEX idx_skills_search ON skills USING gin (search_vector);
CREATE INDEX idx_skills_details ON skills USING gin (details jsonb_path_ops);
-- Specialized index for ESCO URI lookups
CREATE INDEX idx_skills_esco_uri ON skills ((details->'esco'->>'uri'))
    WHERE details->'esco'->>'uri' IS NOT NULL;

-- =============================================================
-- SKILL RELATIONSHIPS
-- =============================================================
CREATE TABLE skill_relationships (
    source_skill_id UUID NOT NULL REFERENCES skills(skill_id) ON DELETE CASCADE,
    target_skill_id UUID NOT NULL REFERENCES skills(skill_id) ON DELETE CASCADE,
    relationship_type VARCHAR(30) NOT NULL,
    -- JSONB for relationship metadata
    metadata        JSONB NOT NULL DEFAULT '{}',    -- {"strength": 0.8, "source": "ai_inferred", "confidence": 0.92}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (source_skill_id, target_skill_id, relationship_type)
);

-- =============================================================
-- JOB ROLES
-- =============================================================
CREATE TABLE job_roles (
    role_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            VARCHAR(255) NOT NULL,
    level           VARCHAR(50),
    job_family      VARCHAR(100),
    -- JSONB for role metadata: O*NET mapping, responsibilities, career paths
    details         JSONB NOT NULL DEFAULT '{
        "description": null,
        "onet_code": null,
        "responsibilities": [],
        "career_progression": {
            "next_roles": [],
            "prerequisite_roles": []
        },
        "typical_skills_count": null,
        "benchmark_data": {}
    }',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE job_role_skills (
    role_id         UUID NOT NULL REFERENCES job_roles(role_id) ON DELETE CASCADE,
    skill_id        UUID NOT NULL REFERENCES skills(skill_id) ON DELETE CASCADE,
    required_level  INTEGER NOT NULL CHECK (required_level >= 1 AND required_level <= 10),
    importance      VARCHAR(20) NOT NULL DEFAULT 'required',
    PRIMARY KEY (role_id, skill_id)
);

-- =============================================================
-- USER SKILLS
-- =============================================================
CREATE TABLE user_skills (
    user_skill_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    skill_id        UUID NOT NULL REFERENCES skills(skill_id) ON DELETE CASCADE,
    current_level   INTEGER NOT NULL,
    target_level    INTEGER,
    assessment_method VARCHAR(30) NOT NULL,
    assessed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- JSONB for assessment evidence and history
    evidence        JSONB NOT NULL DEFAULT '{
        "assessed_by": null,
        "evidence_url": null,
        "evidence_type": null,
        "notes": null,
        "ai_inference": {
            "model_version": null,
            "confidence": null,
            "artifact_sources": [],
            "reasoning": null
        },
        "history": []
    }',
    expires_at      TIMESTAMPTZ,
    UNIQUE (user_id, skill_id, assessment_method)
);

CREATE INDEX idx_user_skills_user ON user_skills (user_id);
CREATE INDEX idx_user_skills_skill ON user_skills (skill_id);

-- =============================================================
-- SKILL GAP ANALYSES
-- =============================================================
CREATE TABLE skill_gap_analyses (
    analysis_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    target_type     VARCHAR(20) NOT NULL,
    target_id       UUID NOT NULL,
    job_role_id     UUID REFERENCES job_roles(role_id),
    overall_readiness_pct NUMERIC(5,2),
    -- JSONB for the full gap analysis detail — variable by analysis type
    analysis_data   JSONB NOT NULL DEFAULT '{
        "gaps": [],
        "strengths": [],
        "recommendations": [],
        "methodology": null,
        "generated_by": "system",
        "model_version": null,
        "comparison_benchmark": null
    }',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_skill_gap_tenant ON skill_gap_analyses (tenant_id);
CREATE INDEX idx_skill_gap_target ON skill_gap_analyses (target_type, target_id);
```

### Domain 3: Learning Content

```sql
-- =============================================================
-- CONTENT PROVIDERS
-- =============================================================
CREATE TABLE content_providers (
    provider_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            VARCHAR(255) NOT NULL,
    provider_type   VARCHAR(30) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- JSONB for provider-specific configuration (varies dramatically by type)
    config          JSONB NOT NULL DEFAULT '{
        "base_url": null,
        "auth": {
            "type": null,
            "api_key": null,
            "oauth": {
                "client_id": null,
                "client_secret": null,
                "token_url": null,
                "scopes": []
            }
        },
        "lti": {
            "client_id": null,
            "deployment_id": null,
            "platform_id": null,
            "auth_login_url": null,
            "auth_token_url": null,
            "keyset_url": null,
            "public_key_pem": null
        },
        "sync": {
            "enabled": false,
            "frequency_hours": 24,
            "last_sync_at": null,
            "catalog_endpoint": null
        },
        "content_mapping": {
            "default_language": "en",
            "skill_auto_map": true
        }
    }',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =============================================================
-- COURSES
-- =============================================================
CREATE TABLE courses (
    course_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    provider_id     UUID REFERENCES content_providers(provider_id),
    -- Relational columns: frequently filtered and sorted
    title           VARCHAR(500) NOT NULL,
    content_type    VARCHAR(30) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    difficulty_level VARCHAR(20),
    language        VARCHAR(10) NOT NULL DEFAULT 'en',
    is_compliance   BOOLEAN NOT NULL DEFAULT FALSE,
    quality_score   NUMERIC(3,2),
    published_at    TIMESTAMPTZ,
    created_by      UUID REFERENCES users(user_id),
    -- JSONB for all variable course metadata
    metadata        JSONB NOT NULL DEFAULT '{
        "description": null,
        "short_description": null,
        "thumbnail_url": null,
        "external_course_id": null,
        "version": "1.0",
        "estimated_duration_minutes": null,
        "tags": [],
        "categories": [],
        "prerequisites": [],
        "learning_objectives": [],
        "target_audience": [],
        "compliance": {
            "category": null,
            "regulation": null,
            "recertification_days": null,
            "passing_score": null,
            "max_attempts": null
        },
        "content_specific": {},
        "quality": {
            "freshness_date": null,
            "avg_learner_rating": null,
            "rating_count": 0,
            "completion_rate": null,
            "source_credibility": null,
            "ai_quality_factors": {}
        },
        "ai_generation": {
            "generated": false,
            "source_document_url": null,
            "prompt_template": null,
            "model_version": null,
            "generated_at": null
        }
    }',
    -- Full-text search
    search_vector   TSVECTOR GENERATED ALWAYS AS (
        to_tsvector('english',
            title || ' ' ||
            COALESCE(metadata->>'description', '') || ' ' ||
            COALESCE(metadata->>'short_description', '')
        )
    ) STORED,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_courses_tenant_status ON courses (tenant_id, status);
CREATE INDEX idx_courses_type ON courses (content_type);
CREATE INDEX idx_courses_compliance ON courses (tenant_id, is_compliance) WHERE is_compliance = TRUE;
CREATE INDEX idx_courses_search ON courses USING gin (search_vector);
CREATE INDEX idx_courses_metadata ON courses USING gin (metadata jsonb_path_ops);
-- Extract commonly filtered JSONB paths into expression indexes
CREATE INDEX idx_courses_tags ON courses USING gin ((metadata->'tags'));
CREATE INDEX idx_courses_difficulty ON courses (difficulty_level);

-- =============================================================
-- COURSE MODULES
-- =============================================================
CREATE TABLE course_modules (
    module_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id       UUID NOT NULL REFERENCES courses(course_id) ON DELETE CASCADE,
    parent_module_id UUID REFERENCES course_modules(module_id),
    title           VARCHAR(500) NOT NULL,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    is_required     BOOLEAN NOT NULL DEFAULT TRUE,
    -- JSONB for module-level configuration
    config          JSONB NOT NULL DEFAULT '{
        "description": null,
        "estimated_duration_minutes": null,
        "completion_criteria": "all_activities",
        "unlock_conditions": [],
        "content_items": []
    }',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_course_modules_course ON course_modules (course_id);

-- =============================================================
-- CONTENT ASSETS
-- =============================================================
CREATE TABLE content_assets (
    asset_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    module_id       UUID REFERENCES course_modules(module_id) ON DELETE SET NULL,
    asset_type      VARCHAR(30) NOT NULL,
    storage_url     TEXT NOT NULL,
    -- JSONB for asset-type-specific metadata
    -- SCORM: {"version": "2004_4th", "entry_point": "index.html", "manifest": {...}, "sco_list": [...]}
    -- Video: {"duration_seconds": 1200, "resolution": "1080p", "transcript": "...", "chapters": [...]}
    -- PDF: {"page_count": 25, "toc": [...], "extracted_text": "..."}
    -- Quiz: {"questions": [...], "time_limit": 30, "shuffle": true}
    asset_metadata  JSONB NOT NULL DEFAULT '{
        "file_name": null,
        "file_size_bytes": null,
        "mime_type": null,
        "checksum_sha256": null,
        "storage_bucket": null,
        "processing_status": "pending",
        "type_specific": {}
    }',
    uploaded_by     UUID REFERENCES users(user_id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_content_assets_tenant ON content_assets (tenant_id);
CREATE INDEX idx_content_assets_module ON content_assets (module_id);
CREATE INDEX idx_content_assets_type ON content_assets (asset_type);

-- =============================================================
-- COURSE-SKILL MAPPINGS
-- =============================================================
CREATE TABLE course_skills (
    course_id       UUID NOT NULL REFERENCES courses(course_id) ON DELETE CASCADE,
    skill_id        UUID NOT NULL REFERENCES skills(skill_id) ON DELETE CASCADE,
    skill_level_taught INTEGER NOT NULL,
    is_primary      BOOLEAN NOT NULL DEFAULT FALSE,
    -- JSONB for mapping metadata
    mapping_data    JSONB NOT NULL DEFAULT '{"source": "manual", "confidence": 1.0}',
    PRIMARY KEY (course_id, skill_id)
);

-- =============================================================
-- ASSESSMENTS (heavily JSONB — question types are extremely variable)
-- =============================================================
CREATE TABLE assessments (
    assessment_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id       UUID NOT NULL REFERENCES courses(course_id) ON DELETE CASCADE,
    module_id       UUID REFERENCES course_modules(module_id),
    title           VARCHAR(500) NOT NULL,
    assessment_type VARCHAR(30) NOT NULL,
    -- JSONB for the entire assessment structure
    -- This is where the hybrid model really shines: question types, scoring rules,
    -- and grading criteria vary enormously and evolve frequently
    definition      JSONB NOT NULL DEFAULT '{
        "passing_score": 70,
        "time_limit_minutes": null,
        "max_attempts": null,
        "shuffle_questions": false,
        "show_feedback": true,
        "is_proctored": false,
        "grading_rubric": null,
        "questions": []
    }',
    -- Example questions array structure (stored in definition.questions):
    -- [
    --   {
    --     "question_id": "uuid",
    --     "type": "multiple_choice",
    --     "text": "What is...?",
    --     "points": 1.0,
    --     "difficulty": "medium",
    --     "skill_id": "uuid",
    --     "options": [
    --       {"id": "a", "text": "Option A", "is_correct": false},
    --       {"id": "b", "text": "Option B", "is_correct": true}
    --     ],
    --     "feedback": {"correct": "Yes!", "incorrect": "Review section 3."},
    --     "ai_generated": false,
    --     "source_document_url": null
    --   },
    --   {
    --     "question_id": "uuid",
    --     "type": "simulation",
    --     "text": "Configure the firewall...",
    --     "points": 5.0,
    --     "simulation_config": {
    --       "environment": "network_lab",
    --       "expected_state": {...},
    --       "grading_script": "..."
    --     }
    --   }
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_assessments_course ON assessments (course_id);
```

### Domain 4: Learning Delivery & Tracking

```sql
-- =============================================================
-- LEARNING PATHWAYS
-- =============================================================
CREATE TABLE learning_pathways (
    pathway_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    title           VARCHAR(500) NOT NULL,
    pathway_type    VARCHAR(30) NOT NULL,
    target_role_id  UUID REFERENCES job_roles(role_id),
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    created_by      UUID REFERENCES users(user_id),
    -- JSONB for pathway structure and AI generation metadata
    definition      JSONB NOT NULL DEFAULT '{
        "description": null,
        "estimated_duration_days": null,
        "steps": [],
        "milestones": [],
        "completion_criteria": "all_required_steps",
        "ai_generation": {
            "generated": false,
            "skill_gaps_addressed": [],
            "model_version": null,
            "optimization_score": null,
            "alternative_pathways_considered": 0
        }
    }',
    -- Example steps structure (stored in definition.steps):
    -- [
    --   {
    --     "step_id": "uuid",
    --     "sort_order": 1,
    --     "type": "course",
    --     "course_id": "uuid",
    --     "title": "Introduction to Leadership",
    --     "is_required": true,
    --     "prerequisite_step_ids": [],
    --     "estimated_duration_minutes": 120,
    --     "unlock_after_days": 0
    --   },
    --   {
    --     "step_id": "uuid",
    --     "sort_order": 2,
    --     "type": "milestone",
    --     "title": "Manager approval checkpoint",
    --     "approval_required": true,
    --     "approver_role": "manager"
    --   }
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pathways_tenant ON learning_pathways (tenant_id);
CREATE INDEX idx_pathways_status ON learning_pathways (tenant_id, status);

-- =============================================================
-- ENROLMENTS
-- =============================================================
CREATE TABLE enrolments (
    enrolment_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    user_id         UUID NOT NULL REFERENCES users(user_id),
    course_id       UUID NOT NULL REFERENCES courses(course_id),
    pathway_id      UUID REFERENCES learning_pathways(pathway_id),
    -- Relational columns: high-cardinality queries, reporting, compliance filters
    enrolment_type  VARCHAR(30) NOT NULL DEFAULT 'voluntary',
    status          VARCHAR(20) NOT NULL DEFAULT 'enrolled',
    due_date        DATE,
    enrolled_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    score           NUMERIC(5,2),
    completion_pct  NUMERIC(5,2) NOT NULL DEFAULT 0,
    time_spent_seconds INTEGER NOT NULL DEFAULT 0,
    attempt_count   INTEGER NOT NULL DEFAULT 0,
    -- JSONB for enrolment context and progress details
    context         JSONB NOT NULL DEFAULT '{
        "assigned_by": null,
        "assigned_reason": null,
        "compliance_rule_id": null,
        "certificate_id": null,
        "certificate_url": null,
        "certificate_number": null,
        "module_progress": {},
        "custom_fields": {},
        "learning_plan_notes": null
    }',
    -- Example module_progress structure (stored in context.module_progress):
    -- {
    --   "mod-uuid-1": {"status": "completed", "pct": 100, "time_seconds": 1200, "completed_at": "..."},
    --   "mod-uuid-2": {"status": "in_progress", "pct": 45, "time_seconds": 600, "last_accessed": "..."},
    --   "mod-uuid-3": {"status": "not_started", "pct": 0, "time_seconds": 0}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_enrolments_user ON enrolments (user_id);
CREATE INDEX idx_enrolments_course ON enrolments (course_id);
CREATE INDEX idx_enrolments_tenant_status ON enrolments (tenant_id, status);
CREATE INDEX idx_enrolments_compliance ON enrolments (due_date)
    WHERE status NOT IN ('completed', 'withdrawn');
CREATE INDEX idx_enrolments_context ON enrolments USING gin (context jsonb_path_ops);

-- =============================================================
-- SCORM RUNTIME DATA (JSONB is ideal here — SCORM data model is a key-value bag)
-- =============================================================
CREATE TABLE scorm_runtime (
    runtime_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    enrolment_id    UUID NOT NULL REFERENCES enrolments(enrolment_id) ON DELETE CASCADE,
    sco_identifier  VARCHAR(255) NOT NULL,
    attempt_number  INTEGER NOT NULL DEFAULT 1,
    -- Core SCORM fields as relational columns (queried for reporting)
    lesson_status   VARCHAR(30),
    score_scaled    NUMERIC(5,4),
    score_raw       NUMERIC(10,4),
    total_time      VARCHAR(50),
    -- The ENTIRE SCORM CMI data model as JSONB
    -- This avoids 50+ nullable columns for SCORM 2004's complex data model
    cmi_data        JSONB NOT NULL DEFAULT '{
        "score_min": null,
        "score_max": null,
        "session_time": null,
        "lesson_location": null,
        "suspend_data": null,
        "entry": null,
        "exit": null,
        "credit": null,
        "mode": null,
        "comments_from_learner": [],
        "comments_from_lms": [],
        "interactions": [],
        "objectives": [],
        "learner_preference": {
            "audio_level": 1.0,
            "language": null,
            "delivery_speed": 1.0,
            "audio_captioning": 0
        },
        "adl": {
            "nav": {
                "request": null,
                "request_valid": {}
            }
        }
    }',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (enrolment_id, sco_identifier, attempt_number)
);

CREATE INDEX idx_scorm_runtime_enrolment ON scorm_runtime (enrolment_id);

-- =============================================================
-- xAPI STATEMENTS (hybrid: relational envelope + JSONB statement body)
-- =============================================================
CREATE TABLE xapi_statements (
    statement_id    UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    -- Extracted relational columns for efficient querying
    actor_user_id   UUID REFERENCES users(user_id),
    verb_id         VARCHAR(500) NOT NULL,
    object_id       VARCHAR(500) NOT NULL,
    object_type     VARCHAR(50) NOT NULL,
    result_success  BOOLEAN,
    result_completion BOOLEAN,
    result_score_scaled NUMERIC(5,4),
    context_registration UUID,
    timestamp       TIMESTAMPTZ NOT NULL,
    stored          TIMESTAMPTZ NOT NULL DEFAULT now(),
    voided          BOOLEAN NOT NULL DEFAULT FALSE,
    -- FULL xAPI statement as JSONB (spec-compliant, nothing lost)
    -- Includes all extensions, context activities, attachments metadata
    statement       JSONB NOT NULL,
    -- Example statement JSONB:
    -- {
    --   "actor": {"mbox": "mailto:jsmith@co.com", "name": "Jane Smith"},
    --   "verb": {"id": "http://adlnet.gov/expapi/verbs/completed", "display": {"en-US": "completed"}},
    --   "object": {"id": "https://lms/course/123", "definition": {...}},
    --   "result": {"score": {...}, "success": true, "completion": true, "duration": "PT2H"},
    --   "context": {"registration": "...", "extensions": {"custom:org-unit": "engineering"}},
    --   "timestamp": "2026-05-20T10:00:00Z"
    -- }
    -- This preserves FULL xAPI fidelity including arbitrary extensions
    -- while the extracted columns enable fast SQL queries
    CONSTRAINT xapi_statement_valid CHECK (
        statement ? 'actor' AND
        statement ? 'verb' AND
        statement ? 'object'
    )
) PARTITION BY RANGE (stored);

-- Monthly partitions
CREATE TABLE xapi_statements_2026_05 PARTITION OF xapi_statements
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE xapi_statements_2026_06 PARTITION OF xapi_statements
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');

CREATE INDEX idx_xapi_tenant_time ON xapi_statements (tenant_id, stored DESC);
CREATE INDEX idx_xapi_actor ON xapi_statements (actor_user_id, stored DESC);
CREATE INDEX idx_xapi_verb ON xapi_statements (verb_id);
CREATE INDEX idx_xapi_object ON xapi_statements (object_id);
CREATE INDEX idx_xapi_registration ON xapi_statements (context_registration);
-- GIN index on full statement for extension queries
CREATE INDEX idx_xapi_statement_gin ON xapi_statements USING gin (statement jsonb_path_ops);

-- =============================================================
-- CERTIFICATIONS
-- =============================================================
CREATE TABLE certifications (
    certification_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            VARCHAR(500) NOT NULL,
    -- JSONB for certification rules and display config
    config          JSONB NOT NULL DEFAULT '{
        "description": null,
        "issuing_body": null,
        "validity_days": null,
        "renewal_required": false,
        "renewal_courses": [],
        "badge_image_url": null,
        "certificate_template": null
    }',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_certifications (
    user_cert_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    certification_id UUID NOT NULL REFERENCES certifications(certification_id),
    enrolment_id    UUID REFERENCES enrolments(enrolment_id),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    issued_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ,
    -- JSONB for certificate details
    details         JSONB NOT NULL DEFAULT '{
        "certificate_number": null,
        "certificate_url": null,
        "verification_code": null,
        "renewal_reminder_sent": false,
        "digital_badge_url": null
    }',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_user_certs_user ON user_certifications (user_id);
CREATE INDEX idx_user_certs_expiry ON user_certifications (expires_at) WHERE status = 'active';

-- =============================================================
-- COMPLIANCE RULES
-- =============================================================
CREATE TABLE compliance_rules (
    rule_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            VARCHAR(255) NOT NULL,
    course_id       UUID NOT NULL REFERENCES courses(course_id),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- JSONB for rule configuration (complex targeting, escalation, notification rules)
    rule_config     JSONB NOT NULL DEFAULT '{
        "description": null,
        "target": {
            "type": "all_users",
            "org_ids": [],
            "role_ids": [],
            "user_group_query": null
        },
        "schedule": {
            "frequency": "annual",
            "frequency_days": 365,
            "anchor_date": null,
            "grace_period_days": 30
        },
        "escalation": {
            "levels": [
                {"days_overdue": 0, "notify": ["learner"], "channel": ["email", "in_app"]},
                {"days_overdue": 7, "notify": ["learner", "manager"], "channel": ["email", "slack"]},
                {"days_overdue": 14, "notify": ["learner", "manager", "hr"], "channel": ["email"]}
            ]
        },
        "audit": {
            "require_acknowledgement": false,
            "export_format": "csv",
            "retention_years": 7
        }
    }',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_rules_tenant ON compliance_rules (tenant_id);
```

### Domain 5: Analytics, ROI & Audit

```sql
-- =============================================================
-- LEARNING EVENTS (analytics event stream)
-- =============================================================
CREATE TABLE learning_events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID NOT NULL,
    event_type      VARCHAR(50) NOT NULL,
    course_id       UUID,
    skill_id        UUID,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- JSONB for event-type-specific payload (avoids a wide table)
    payload         JSONB NOT NULL DEFAULT '{}',
    -- Example payloads by event_type:
    -- 'course_completed': {"score": 92, "time_spent_seconds": 14400, "attempt": 1}
    -- 'skill_level_changed': {"from": 3, "to": 5, "method": "assessment"}
    -- 'nudge_clicked': {"channel": "slack", "nudge_type": "learning_reminder", "course_id": "..."}
    -- 'content_rated': {"rating": 4.5, "comment": "Very helpful!", "course_id": "..."}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

CREATE TABLE learning_events_2026_05 PARTITION OF learning_events
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_learning_events_tenant ON learning_events (tenant_id, occurred_at DESC);
CREATE INDEX idx_learning_events_user ON learning_events (user_id, occurred_at DESC);
CREATE INDEX idx_learning_events_type ON learning_events (event_type, occurred_at DESC);

-- =============================================================
-- BUSINESS KPI DATA
-- =============================================================
CREATE TABLE business_kpis (
    kpi_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            VARCHAR(255) NOT NULL,
    -- JSONB for KPI definition and import configuration
    definition      JSONB NOT NULL DEFAULT '{
        "description": null,
        "category": null,
        "unit": null,
        "higher_is_better": true,
        "data_source": null,
        "import_config": {
            "type": "manual",
            "api_endpoint": null,
            "field_mapping": {},
            "sync_frequency_hours": null
        },
        "thresholds": {
            "poor": null,
            "acceptable": null,
            "good": null,
            "excellent": null
        }
    }',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE business_kpi_values (
    value_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    kpi_id          UUID NOT NULL REFERENCES business_kpis(kpi_id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(user_id),
    org_id          UUID REFERENCES organizations(org_id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    value           NUMERIC(15,4) NOT NULL,
    -- JSONB for supplementary data points
    context         JSONB NOT NULL DEFAULT '{}',    -- {"source": "salesforce", "raw_data": {...}}
    imported_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_kpi_values_kpi ON business_kpi_values (kpi_id, period_start);
CREATE INDEX idx_kpi_values_user ON business_kpi_values (user_id, period_start);

-- =============================================================
-- ROI REPORTS
-- =============================================================
CREATE TABLE roi_reports (
    report_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    title           VARCHAR(500) NOT NULL,
    report_type     VARCHAR(30) NOT NULL,
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    generated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    generated_by    UUID REFERENCES users(user_id),
    status          VARCHAR(20) NOT NULL DEFAULT 'generated',
    -- JSONB for the entire report content (complex, variable structure)
    report_data     JSONB NOT NULL DEFAULT '{
        "summary": {
            "total_learning_hours": 0,
            "total_completions": 0,
            "total_certifications": 0,
            "active_learners": 0,
            "compliance_rate": null
        },
        "correlations": [],
        "pathway_effectiveness": [],
        "content_quality_rankings": [],
        "recommendations": [],
        "flagged_pathways": [],
        "methodology": null
    }'
    -- Example correlations structure:
    -- [
    --   {
    --     "kpi_id": "...",
    --     "kpi_name": "Sales Conversion",
    --     "course_id": "...",
    --     "course_title": "Advanced Negotiation",
    --     "learner_cohort": 45,
    --     "control_cohort": 52,
    --     "kpi_before": 12.3,
    --     "kpi_after": 15.7,
    --     "delta_pct": 27.6,
    --     "confidence": 0.94,
    --     "significant": true,
    --     "attribution_method": "control_group"
    --   }
    -- ]
);

-- =============================================================
-- NOTIFICATIONS
-- =============================================================
CREATE TABLE notifications (
    notification_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID NOT NULL REFERENCES users(user_id),
    channel         VARCHAR(20) NOT NULL,
    notification_type VARCHAR(50) NOT NULL,
    is_read         BOOLEAN NOT NULL DEFAULT FALSE,
    sent_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    read_at         TIMESTAMPTZ,
    -- JSONB for notification content (varies by type and channel)
    content         JSONB NOT NULL DEFAULT '{
        "title": null,
        "body": null,
        "action_url": null,
        "template": null,
        "template_vars": {},
        "channel_specific": {}
    }'
);

CREATE INDEX idx_notifications_user ON notifications (user_id, sent_at DESC);
CREATE INDEX idx_notifications_unread ON notifications (user_id) WHERE is_read = FALSE;

-- =============================================================
-- AUDIT LOG
-- =============================================================
CREATE TABLE audit_log (
    log_id          BIGSERIAL PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    user_id         UUID,
    action          VARCHAR(50) NOT NULL,
    resource_type   VARCHAR(100) NOT NULL,
    resource_id     UUID,
    -- JSONB for change diff (old_values and new_values can be any shape)
    changes         JSONB NOT NULL DEFAULT '{
        "old": null,
        "new": null
    }',
    -- JSONB for request context
    request_context JSONB NOT NULL DEFAULT '{
        "ip_address": null,
        "user_agent": null,
        "session_id": null,
        "api_key_id": null
    }',
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

CREATE TABLE audit_log_2026_05 PARTITION OF audit_log
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_audit_log_tenant ON audit_log (tenant_id, occurred_at DESC);
CREATE INDEX idx_audit_log_user ON audit_log (user_id, occurred_at DESC);
CREATE INDEX idx_audit_log_resource ON audit_log (resource_type, resource_id);

-- =============================================================
-- INTEGRATION SYNC LOG
-- =============================================================
CREATE TABLE integration_sync_log (
    sync_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    integration_type VARCHAR(30) NOT NULL,
    direction       VARCHAR(10) NOT NULL,
    status          VARCHAR(20) NOT NULL,
    records_processed INTEGER NOT NULL DEFAULT 0,
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    -- JSONB for sync details and error info
    details         JSONB NOT NULL DEFAULT '{
        "records_created": 0,
        "records_updated": 0,
        "records_deleted": 0,
        "records_failed": 0,
        "errors": [],
        "warnings": [],
        "config_snapshot": {}
    }'
);

CREATE INDEX idx_sync_log_tenant ON integration_sync_log (tenant_id, started_at DESC);
```

### Useful Queries Demonstrating the Hybrid Approach

```sql
-- Query 1: Find courses tagged with a specific skill, filtered by relational columns,
-- with details from JSONB
SELECT
    c.course_id,
    c.title,
    c.difficulty_level,
    c.quality_score,
    c.metadata->>'description' AS description,
    c.metadata->'quality'->>'avg_learner_rating' AS rating,
    c.metadata->'compliance'->>'category' AS compliance_category,
    s.name AS skill_name,
    cs.skill_level_taught
FROM courses c
JOIN course_skills cs ON cs.course_id = c.course_id
JOIN skills s ON s.skill_id = cs.skill_id
WHERE c.tenant_id = $1
    AND c.status = 'published'                     -- relational filter (indexed)
    AND c.is_compliance = FALSE                    -- relational filter (indexed)
    AND s.skill_type = 'technical'                 -- relational filter (indexed)
    AND c.metadata @> '{"tags": ["leadership"]}'   -- JSONB containment (GIN indexed)
ORDER BY c.quality_score DESC NULLS LAST;

-- Query 2: User skill profile with ESCO URIs and assessment evidence
SELECT
    u.display_name,
    s.name AS skill_name,
    s.details->'esco'->>'uri' AS esco_uri,
    us.current_level,
    us.target_level,
    us.assessment_method,
    (s.details->'proficiency_scale'->'levels'->(us.current_level - 1)->>'label') AS level_label,
    us.evidence->>'evidence_url' AS evidence,
    us.evidence->'ai_inference'->>'confidence' AS ai_confidence,
    jsonb_array_length(us.evidence->'history') AS assessment_count
FROM user_skills us
JOIN users u ON u.user_id = us.user_id
JOIN skills s ON s.skill_id = us.skill_id
WHERE us.user_id = $1
ORDER BY us.current_level DESC;

-- Query 3: Compliance status leveraging JSONB rule configuration
SELECT
    cr.name AS rule_name,
    cr.rule_config->'schedule'->>'frequency' AS frequency,
    u.display_name,
    e.status AS enrolment_status,
    e.completed_at,
    e.due_date,
    CASE
        WHEN e.enrolment_id IS NULL THEN 'not_enrolled'
        WHEN e.status = 'completed' THEN 'compliant'
        WHEN e.due_date < now() THEN 'overdue'
        ELSE 'in_progress'
    END AS compliance_status,
    -- Get escalation level from JSONB config
    (SELECT jsonb_agg(level)
     FROM jsonb_array_elements(cr.rule_config->'escalation'->'levels') AS level
     WHERE (level->>'days_overdue')::int <= EXTRACT(DAY FROM now() - e.due_date)
    ) AS triggered_escalations
FROM compliance_rules cr
CROSS JOIN users u
LEFT JOIN enrolments e ON e.user_id = u.user_id AND e.course_id = cr.course_id
WHERE cr.tenant_id = $1 AND cr.is_active = TRUE AND u.status = 'active';

-- Query 4: xAPI query using both relational and JSONB indexes
SELECT
    xs.statement_id,
    xs.verb_id,
    xs.result_score_scaled,
    xs.timestamp,
    xs.statement->'actor'->>'name' AS actor_name,
    xs.statement->'object'->'definition'->'name'->>'en-US' AS activity_name,
    xs.statement->'result'->'score'->>'raw' AS raw_score,
    xs.statement->'context'->'extensions' AS custom_extensions
FROM xapi_statements xs
WHERE xs.tenant_id = $1
    AND xs.actor_user_id = $2                      -- relational filter (indexed)
    AND xs.verb_id = 'http://adlnet.gov/expapi/verbs/completed'  -- relational (indexed)
    AND xs.stored > now() - interval '30 days'     -- partition pruning
ORDER BY xs.timestamp DESC
LIMIT 50;
```

---

## Pros and Cons

### Pros

1. **Single database technology**: No need for a separate document store, event store, or graph database. PostgreSQL handles everything, simplifying operations, backups, and deployment.

2. **Schema evolution without migrations**: Adding new fields to skills, assessments, course metadata, or compliance rules is a JSONB update — no DDL migration, no downtime, no column additions. This is crucial for a platform that will evolve rapidly during its first years.

3. **Full xAPI fidelity**: Storing the complete xAPI statement as JSONB preserves every extension, custom context, and nested structure from content providers. The extracted relational columns provide fast query performance for the most common access patterns.

4. **Tenant customization**: Custom fields per tenant are stored in JSONB columns (user.profile.custom_fields, enrolment.context.custom_fields). No per-tenant schema modifications needed.

5. **Best-of-both query patterns**: Compliance dashboards use relational JOINs and filters for speed. Content search uses full-text search vectors. xAPI analytics can use either relational indexes for common queries or JSONB GIN indexes for extension-specific queries.

6. **SCORM compatibility**: The SCORM CMI data model maps naturally to a JSONB column. SCORM's key-value pair structure (cmi.core.lesson_status, cmi.core.score.raw, etc.) fits perfectly into a JSON document without the dozens of nullable columns a fully normalized approach requires.

7. **Assessment flexibility**: Question types, scoring rubrics, and grading criteria are stored as JSONB. Adding a new question type (e.g., AI-graded code review) requires no schema change — just a new shape in the questions array.

8. **Reasonable team skill requirements**: Most backend developers are comfortable with SQL and JSON. The hybrid pattern does not require specialized event sourcing or graph database expertise.

### Cons

1. **JSONB query performance**: Complex queries into deeply nested JSONB structures can be slower than equivalent relational queries. GIN indexes help, but they consume significant disk space (sometimes 2-3x the data size).

2. **No referential integrity inside JSONB**: If a JSONB column references a skill_id or course_id, the database cannot enforce that the referenced entity exists. Application-level validation is required.

3. **Schema discipline required**: Without enforcement, JSONB columns can become dumping grounds for unstructured data. Teams must maintain JSON Schema definitions and validate at the application layer.

4. **Indexing trade-offs**: GIN indexes on large JSONB columns are expensive to maintain and can slow write performance. Expression indexes on specific paths (e.g., `(metadata->'tags')`) are more targeted but must be manually created as query patterns emerge.

5. **ORM friction**: While modern ORMs support JSONB, the mapping is not as clean as pure relational columns. Type safety for JSONB content requires additional application-layer types or code generation.

6. **Reporting complexity**: Business intelligence tools and report builders work best with flat, relational schemas. JSONB data may need to be unpacked into views or materialized tables for reporting tools like Metabase, Looker, or Tableau.

7. **Data migration complexity**: When a JSONB field needs to be promoted to a relational column (because query patterns changed), the migration requires reading every row, extracting the field, and backfilling — potentially expensive for large tables.

8. **Testing overhead**: Test fixtures must construct valid JSONB payloads. Schema changes in JSONB structures are harder to detect through database-level tests compared to relational DDL changes.

---

## Migration and Scaling Considerations

### Migration Strategy

1. **Progressive JSONB enrichment**: Start with minimal JSONB defaults. As the platform discovers what tenant-specific data looks like in production, evolve the JSONB shapes. No DDL migration needed.

2. **Promotion path**: When a JSONB field is queried frequently enough to warrant a relational column:
   ```sql
   -- Step 1: Add the relational column
   ALTER TABLE courses ADD COLUMN avg_rating NUMERIC(3,2);
   
   -- Step 2: Backfill from JSONB (can run concurrently)
   UPDATE courses SET avg_rating = (metadata->'quality'->>'avg_learner_rating')::numeric;
   
   -- Step 3: Create index
   CREATE INDEX CONCURRENTLY idx_courses_rating ON courses (avg_rating);
   
   -- Step 4: Update application to write both relational column and JSONB
   -- Step 5: Migrate read queries to use relational column
   ```

3. **JSON Schema registry**: Maintain a version-controlled registry of JSON Schemas for each JSONB column. Validate on write at the application layer. This provides documentation and catches breaking changes.

### Scaling Strategy

1. **Partitioning**: xapi_statements, learning_events, and audit_log are partitioned by month. This provides automatic partition pruning for time-range queries.

2. **Read replicas**: Deploy PostgreSQL streaming replicas for analytics, reporting, and materialized view refresh.

3. **JSONB size monitoring**: Monitor average JSONB column sizes. If any column regularly exceeds 10KB per row, consider normalizing the heaviest sub-documents into separate tables.

4. **GIN index maintenance**: GIN indexes require periodic `REINDEX CONCURRENTLY` to reclaim space. Schedule this during maintenance windows.

5. **Selective JSONB indexing**: Do not GIN-index every JSONB column. Only index columns that participate in WHERE clauses. Use expression indexes for specific paths rather than full-column GIN indexes where possible.

6. **Connection pooling**: PgBouncer in transaction mode. JSONB operations are slightly more CPU-intensive than pure relational operations, so monitor PostgreSQL CPU usage and scale vertically or add replicas as needed.

7. **Estimated storage**: For a 50,000-user organization:
   - users table: ~50K rows, ~50MB (including JSONB profiles)
   - enrolments: ~500K rows/year, ~200MB/year
   - xapi_statements: ~180M rows/year, ~100GB/year (with JSONB statements averaging 500 bytes)
   - Total first year: ~120GB, manageable on a single PostgreSQL instance with SSDs

### JSONB Compression

PostgreSQL automatically uses TOAST compression for JSONB values exceeding ~2KB. For xAPI statements and SCORM suspend_data, this can reduce storage by 50-70%. Monitor TOAST table sizes with:

```sql
SELECT
    relname,
    pg_size_pretty(pg_total_relation_size(oid)) AS total_size,
    pg_size_pretty(pg_relation_size(oid)) AS main_size,
    pg_size_pretty(pg_total_relation_size(reltoastrelid)) AS toast_size
FROM pg_class
WHERE relname IN ('xapi_statements', 'scorm_runtime', 'courses');
```
