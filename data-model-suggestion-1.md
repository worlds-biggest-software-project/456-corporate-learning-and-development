# Data Model Suggestion 1: Normalized Relational Database (PostgreSQL)

## Overview

This model uses a fully normalized relational schema in PostgreSQL, following Third Normal Form (3NF) throughout. Every entity has its own table with explicit foreign key relationships, referential integrity constraints, and well-defined indexes. This is the traditional approach for enterprise systems where data consistency, ACID compliance, and audit trail reliability are paramount — all critical requirements for a corporate L&D platform that handles compliance training, certification tracking, and regulatory audit exports.

The schema is organized into six logical domains: Identity & Organization, Skills Taxonomy, Learning Content, Learning Delivery & Tracking, Analytics & ROI, and Integration & Audit.

---

## Technology Recommendations

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Primary Database | PostgreSQL 16+ | Mature, ACID-compliant, excellent extension ecosystem, strong JSON support for future flexibility |
| Connection Pooling | PgBouncer | Transaction-level pooling for high-concurrency web workloads |
| Full-Text Search | PostgreSQL tsvector + pg_trgm | Built-in full-text for course/content search without external dependency |
| Migrations | Flyway or golang-migrate | Version-controlled, repeatable schema migrations |
| Read Replicas | PostgreSQL streaming replication | Offload analytics and reporting queries |
| Backup | pg_basebackup + WAL archiving | Point-in-time recovery for compliance requirements |

---

## Schema Definition

### Domain 1: Identity & Organization

```sql
-- =============================================================
-- TENANTS (multi-tenant support)
-- =============================================================
CREATE TABLE tenants (
    tenant_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
    settings_json   JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenants_slug ON tenants (slug);

-- =============================================================
-- ORGANIZATIONS (departments, business units, teams)
-- =============================================================
CREATE TABLE organizations (
    org_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    parent_org_id   UUID REFERENCES organizations(org_id),
    name            VARCHAR(255) NOT NULL,
    org_type        VARCHAR(50) NOT NULL CHECK (org_type IN ('company', 'division', 'department', 'team')),
    code            VARCHAR(50),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_organizations_tenant ON organizations (tenant_id);
CREATE INDEX idx_organizations_parent ON organizations (parent_org_id);

-- =============================================================
-- USERS
-- =============================================================
CREATE TABLE users (
    user_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    org_id          UUID REFERENCES organizations(org_id),
    external_id     VARCHAR(255),                   -- SCIM externalId
    username        VARCHAR(255) NOT NULL,
    email           VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),                   -- NULL for SSO-only users
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    display_name    VARCHAR(255),
    job_title       VARCHAR(255),
    employee_number VARCHAR(100),
    manager_id      UUID REFERENCES users(user_id),
    hire_date       DATE,
    status          VARCHAR(20) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'suspended', 'pending')),
    last_login_at   TIMESTAMPTZ,
    timezone        VARCHAR(50) DEFAULT 'UTC',
    locale          VARCHAR(10) DEFAULT 'en',
    avatar_url      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, username),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users (tenant_id);
CREATE INDEX idx_users_org ON users (org_id);
CREATE INDEX idx_users_manager ON users (manager_id);
CREATE INDEX idx_users_external_id ON users (tenant_id, external_id);
CREATE INDEX idx_users_status ON users (tenant_id, status);

-- =============================================================
-- ROLES & PERMISSIONS
-- =============================================================
CREATE TABLE roles (
    role_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    is_system_role  BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE permissions (
    permission_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource        VARCHAR(100) NOT NULL,          -- e.g., 'courses', 'users', 'reports'
    action          VARCHAR(50) NOT NULL,            -- e.g., 'read', 'write', 'delete', 'assign'
    description     TEXT,
    UNIQUE (resource, action)
);

CREATE TABLE role_permissions (
    role_id         UUID NOT NULL REFERENCES roles(role_id) ON DELETE CASCADE,
    permission_id   UUID NOT NULL REFERENCES permissions(permission_id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(role_id) ON DELETE CASCADE,
    assigned_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    assigned_by     UUID REFERENCES users(user_id),
    PRIMARY KEY (user_id, role_id)
);

-- =============================================================
-- SSO / IDENTITY PROVIDER CONNECTIONS
-- =============================================================
CREATE TABLE identity_providers (
    idp_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    provider_type   VARCHAR(20) NOT NULL CHECK (provider_type IN ('saml', 'oidc')),
    name            VARCHAR(255) NOT NULL,
    entity_id       VARCHAR(500),                   -- SAML entity ID
    metadata_url    TEXT,                            -- SAML metadata URL
    client_id       VARCHAR(255),                   -- OIDC client_id
    client_secret   VARCHAR(500),                   -- encrypted
    issuer_url      TEXT,                            -- OIDC issuer
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_identity_links (
    link_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    idp_id          UUID NOT NULL REFERENCES identity_providers(idp_id),
    external_subject VARCHAR(500) NOT NULL,         -- sub claim or SAML NameID
    linked_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (idp_id, external_subject)
);
```

### Domain 2: Skills Taxonomy

```sql
-- =============================================================
-- SKILL FRAMEWORKS (ESCO, O*NET, custom)
-- =============================================================
CREATE TABLE skill_frameworks (
    framework_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            VARCHAR(255) NOT NULL,
    source          VARCHAR(50) NOT NULL CHECK (source IN ('esco', 'onet', 'custom', 'imported')),
    version         VARCHAR(50),
    description     TEXT,
    is_default      BOOLEAN NOT NULL DEFAULT FALSE,
    imported_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =============================================================
-- SKILL CATEGORIES (hierarchical grouping)
-- =============================================================
CREATE TABLE skill_categories (
    category_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework_id    UUID NOT NULL REFERENCES skill_frameworks(framework_id) ON DELETE CASCADE,
    parent_id       UUID REFERENCES skill_categories(category_id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50),
    description     TEXT,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    level_depth     INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_skill_categories_framework ON skill_categories (framework_id);
CREATE INDEX idx_skill_categories_parent ON skill_categories (parent_id);

-- =============================================================
-- SKILLS
-- =============================================================
CREATE TABLE skills (
    skill_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework_id    UUID NOT NULL REFERENCES skill_frameworks(framework_id) ON DELETE CASCADE,
    category_id     UUID REFERENCES skill_categories(category_id),
    external_uri    TEXT,                            -- ESCO URI or O*NET element ID
    name            VARCHAR(500) NOT NULL,
    description     TEXT,
    skill_type      VARCHAR(30) NOT NULL CHECK (skill_type IN ('technical', 'soft', 'transversal', 'domain', 'certification')),
    proficiency_scale VARCHAR(30) NOT NULL DEFAULT 'five_level'
        CHECK (proficiency_scale IN ('binary', 'three_level', 'five_level', 'ten_level', 'percentage')),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_skills_framework ON skills (framework_id);
CREATE INDEX idx_skills_category ON skills (category_id);
CREATE INDEX idx_skills_name ON skills USING gin (to_tsvector('english', name));
CREATE INDEX idx_skills_type ON skills (skill_type);

-- =============================================================
-- SKILL RELATIONSHIPS (prerequisite, related, supersedes)
-- =============================================================
CREATE TABLE skill_relationships (
    relationship_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_skill_id UUID NOT NULL REFERENCES skills(skill_id) ON DELETE CASCADE,
    target_skill_id UUID NOT NULL REFERENCES skills(skill_id) ON DELETE CASCADE,
    relationship_type VARCHAR(30) NOT NULL CHECK (relationship_type IN ('prerequisite', 'related', 'supersedes', 'component_of', 'alternative')),
    strength        NUMERIC(3,2) CHECK (strength >= 0 AND strength <= 1),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_skill_id, target_skill_id, relationship_type),
    CHECK (source_skill_id != target_skill_id)
);

CREATE INDEX idx_skill_relationships_source ON skill_relationships (source_skill_id);
CREATE INDEX idx_skill_relationships_target ON skill_relationships (target_skill_id);

-- =============================================================
-- JOB ROLES & ROLE-SKILL MAPPINGS
-- =============================================================
CREATE TABLE job_roles (
    role_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    level           VARCHAR(50),                    -- e.g., 'junior', 'mid', 'senior', 'lead'
    job_family      VARCHAR(100),
    onet_code       VARCHAR(20),                    -- O*NET-SOC code
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE job_role_skills (
    role_id         UUID NOT NULL REFERENCES job_roles(role_id) ON DELETE CASCADE,
    skill_id        UUID NOT NULL REFERENCES skills(skill_id) ON DELETE CASCADE,
    required_level  INTEGER NOT NULL CHECK (required_level >= 1 AND required_level <= 10),
    importance      VARCHAR(20) NOT NULL DEFAULT 'required' CHECK (importance IN ('critical', 'required', 'preferred', 'nice_to_have')),
    PRIMARY KEY (role_id, skill_id)
);

-- =============================================================
-- USER SKILLS (assessed competency levels)
-- =============================================================
CREATE TABLE user_skills (
    user_skill_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    skill_id        UUID NOT NULL REFERENCES skills(skill_id) ON DELETE CASCADE,
    current_level   INTEGER NOT NULL CHECK (current_level >= 0 AND current_level <= 10),
    target_level    INTEGER CHECK (target_level >= 0 AND target_level <= 10),
    assessment_method VARCHAR(30) NOT NULL CHECK (assessment_method IN ('self', 'manager', 'peer', 'ai_inferred', 'assessment', 'certification', 'work_artifact')),
    evidence_url    TEXT,
    assessed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    assessed_by     UUID REFERENCES users(user_id),
    expires_at      TIMESTAMPTZ,
    UNIQUE (user_id, skill_id, assessment_method)
);

CREATE INDEX idx_user_skills_user ON user_skills (user_id);
CREATE INDEX idx_user_skills_skill ON user_skills (skill_id);

-- =============================================================
-- SKILL GAP ANALYSES (snapshots)
-- =============================================================
CREATE TABLE skill_gap_analyses (
    analysis_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    target_type     VARCHAR(20) NOT NULL CHECK (target_type IN ('user', 'team', 'department', 'organization')),
    target_id       UUID NOT NULL,                  -- references user_id or org_id
    job_role_id     UUID REFERENCES job_roles(role_id),
    analysis_date   TIMESTAMPTZ NOT NULL DEFAULT now(),
    overall_readiness_pct NUMERIC(5,2),
    generated_by    VARCHAR(20) NOT NULL DEFAULT 'system' CHECK (generated_by IN ('system', 'manual', 'ai')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE skill_gap_details (
    detail_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    analysis_id     UUID NOT NULL REFERENCES skill_gap_analyses(analysis_id) ON DELETE CASCADE,
    skill_id        UUID NOT NULL REFERENCES skills(skill_id),
    required_level  INTEGER NOT NULL,
    current_level   INTEGER NOT NULL,
    gap_size        INTEGER GENERATED ALWAYS AS (required_level - current_level) STORED,
    priority        VARCHAR(20) NOT NULL DEFAULT 'medium' CHECK (priority IN ('critical', 'high', 'medium', 'low')),
    recommended_action TEXT
);

CREATE INDEX idx_skill_gap_details_analysis ON skill_gap_details (analysis_id);
```

### Domain 3: Learning Content

```sql
-- =============================================================
-- CONTENT PROVIDERS (LinkedIn Learning, Coursera, internal, etc.)
-- =============================================================
CREATE TABLE content_providers (
    provider_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            VARCHAR(255) NOT NULL,
    provider_type   VARCHAR(30) NOT NULL CHECK (provider_type IN ('internal', 'linkedin_learning', 'coursera', 'udemy_business', 'custom_lti', 'scorm_package', 'xapi_package')),
    base_url        TEXT,
    api_key_encrypted VARCHAR(500),
    lti_client_id   VARCHAR(255),
    lti_deployment_id VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
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
    external_course_id VARCHAR(255),                -- ID in external provider
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    short_description VARCHAR(500),
    thumbnail_url   TEXT,
    content_type    VARCHAR(30) NOT NULL CHECK (content_type IN ('scorm_12', 'scorm_2004', 'xapi', 'cmi5', 'lti', 'video', 'document', 'ai_generated', 'blended', 'external_url')),
    difficulty_level VARCHAR(20) CHECK (difficulty_level IN ('beginner', 'intermediate', 'advanced', 'expert')),
    estimated_duration_minutes INTEGER,
    language        VARCHAR(10) NOT NULL DEFAULT 'en',
    version         VARCHAR(50) NOT NULL DEFAULT '1.0',
    status          VARCHAR(20) NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'review', 'published', 'archived', 'retired')),
    is_compliance   BOOLEAN NOT NULL DEFAULT FALSE,
    compliance_category VARCHAR(100),
    recertification_days INTEGER,                   -- days before cert expires
    passing_score   NUMERIC(5,2),                   -- minimum score to pass
    max_attempts    INTEGER,
    tags            TEXT[],                          -- PostgreSQL array for tagging
    quality_score   NUMERIC(3,2),                   -- AI-computed content quality
    freshness_date  DATE,                           -- last content review date
    published_at    TIMESTAMPTZ,
    retired_at      TIMESTAMPTZ,
    created_by      UUID REFERENCES users(user_id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_courses_tenant ON courses (tenant_id);
CREATE INDEX idx_courses_provider ON courses (provider_id);
CREATE INDEX idx_courses_status ON courses (tenant_id, status);
CREATE INDEX idx_courses_compliance ON courses (tenant_id, is_compliance) WHERE is_compliance = TRUE;
CREATE INDEX idx_courses_title_search ON courses USING gin (to_tsvector('english', title || ' ' || COALESCE(description, '')));
CREATE INDEX idx_courses_tags ON courses USING gin (tags);

-- =============================================================
-- COURSE MODULES (sections/chapters within a course)
-- =============================================================
CREATE TABLE course_modules (
    module_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id       UUID NOT NULL REFERENCES courses(course_id) ON DELETE CASCADE,
    parent_module_id UUID REFERENCES course_modules(module_id),
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    estimated_duration_minutes INTEGER,
    is_required     BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_course_modules_course ON course_modules (course_id);

-- =============================================================
-- CONTENT ASSETS (files, SCORM packages, videos, etc.)
-- =============================================================
CREATE TABLE content_assets (
    asset_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    module_id       UUID REFERENCES course_modules(module_id) ON DELETE SET NULL,
    asset_type      VARCHAR(30) NOT NULL CHECK (asset_type IN ('scorm_package', 'xapi_package', 'cmi5_package', 'video', 'pdf', 'document', 'image', 'html', 'quiz', 'assessment')),
    file_name       VARCHAR(500),
    file_size_bytes BIGINT,
    mime_type       VARCHAR(100),
    storage_url     TEXT NOT NULL,                   -- S3 or local path
    storage_bucket  VARCHAR(255),
    checksum_sha256 VARCHAR(64),
    scorm_version   VARCHAR(20),
    scorm_entry_point VARCHAR(500),                 -- e.g. index.html for SCORM
    duration_seconds INTEGER,
    transcript_text TEXT,                            -- for video/audio accessibility
    uploaded_by     UUID REFERENCES users(user_id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_content_assets_tenant ON content_assets (tenant_id);
CREATE INDEX idx_content_assets_module ON content_assets (module_id);

-- =============================================================
-- COURSE-SKILL MAPPINGS
-- =============================================================
CREATE TABLE course_skills (
    course_id       UUID NOT NULL REFERENCES courses(course_id) ON DELETE CASCADE,
    skill_id        UUID NOT NULL REFERENCES skills(skill_id) ON DELETE CASCADE,
    skill_level_taught INTEGER NOT NULL CHECK (skill_level_taught >= 1 AND skill_level_taught <= 10),
    is_primary      BOOLEAN NOT NULL DEFAULT FALSE,
    PRIMARY KEY (course_id, skill_id)
);

CREATE INDEX idx_course_skills_skill ON course_skills (skill_id);

-- =============================================================
-- ASSESSMENTS & QUESTIONS
-- =============================================================
CREATE TABLE assessments (
    assessment_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id       UUID NOT NULL REFERENCES courses(course_id) ON DELETE CASCADE,
    module_id       UUID REFERENCES course_modules(module_id),
    title           VARCHAR(500) NOT NULL,
    assessment_type VARCHAR(30) NOT NULL CHECK (assessment_type IN ('quiz', 'exam', 'practical', 'peer_review', 'self_assessment', 'ai_generated')),
    passing_score   NUMERIC(5,2),
    time_limit_minutes INTEGER,
    max_attempts    INTEGER,
    shuffle_questions BOOLEAN NOT NULL DEFAULT FALSE,
    is_proctored    BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE assessment_questions (
    question_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    assessment_id   UUID NOT NULL REFERENCES assessments(assessment_id) ON DELETE CASCADE,
    question_type   VARCHAR(30) NOT NULL CHECK (question_type IN ('multiple_choice', 'true_false', 'fill_blank', 'essay', 'matching', 'ordering', 'hotspot', 'simulation')),
    question_text   TEXT NOT NULL,
    explanation     TEXT,
    points          NUMERIC(5,2) NOT NULL DEFAULT 1.0,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    skill_id        UUID REFERENCES skills(skill_id),  -- which skill this question tests
    difficulty      VARCHAR(20) CHECK (difficulty IN ('easy', 'medium', 'hard')),
    ai_generated    BOOLEAN NOT NULL DEFAULT FALSE,
    source_document_url TEXT,                       -- compliance doc this was generated from
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE question_options (
    option_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    question_id     UUID NOT NULL REFERENCES assessment_questions(question_id) ON DELETE CASCADE,
    option_text     TEXT NOT NULL,
    is_correct      BOOLEAN NOT NULL DEFAULT FALSE,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    feedback_text   TEXT
);
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
    description     TEXT,
    target_role_id  UUID REFERENCES job_roles(role_id),
    pathway_type    VARCHAR(30) NOT NULL CHECK (pathway_type IN ('manual', 'ai_generated', 'compliance', 'onboarding', 'upskilling', 'reskilling')),
    estimated_duration_days INTEGER,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'active', 'archived')),
    created_by      UUID REFERENCES users(user_id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pathway_steps (
    step_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pathway_id      UUID NOT NULL REFERENCES learning_pathways(pathway_id) ON DELETE CASCADE,
    course_id       UUID REFERENCES courses(course_id),
    assessment_id   UUID REFERENCES assessments(assessment_id),
    step_type       VARCHAR(30) NOT NULL CHECK (step_type IN ('course', 'assessment', 'activity', 'milestone', 'approval')),
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    is_required     BOOLEAN NOT NULL DEFAULT TRUE,
    prerequisite_step_id UUID REFERENCES pathway_steps(step_id),
    estimated_duration_minutes INTEGER
);

CREATE INDEX idx_pathway_steps_pathway ON pathway_steps (pathway_id);

-- =============================================================
-- ENROLMENTS
-- =============================================================
CREATE TABLE enrolments (
    enrolment_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(user_id),
    course_id       UUID NOT NULL REFERENCES courses(course_id),
    pathway_id      UUID REFERENCES learning_pathways(pathway_id),
    enrolment_type  VARCHAR(30) NOT NULL DEFAULT 'voluntary'
        CHECK (enrolment_type IN ('voluntary', 'assigned', 'compliance_required', 'ai_recommended')),
    status          VARCHAR(20) NOT NULL DEFAULT 'enrolled'
        CHECK (status IN ('enrolled', 'in_progress', 'completed', 'failed', 'expired', 'withdrawn')),
    assigned_by     UUID REFERENCES users(user_id),
    assigned_reason TEXT,
    due_date        DATE,
    enrolled_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    expired_at      TIMESTAMPTZ,
    score           NUMERIC(5,2),
    attempt_count   INTEGER NOT NULL DEFAULT 0,
    time_spent_seconds INTEGER NOT NULL DEFAULT 0,
    completion_pct  NUMERIC(5,2) NOT NULL DEFAULT 0,
    certificate_id  UUID,
    UNIQUE (user_id, course_id, enrolled_at)
);

CREATE INDEX idx_enrolments_user ON enrolments (user_id);
CREATE INDEX idx_enrolments_course ON enrolments (course_id);
CREATE INDEX idx_enrolments_status ON enrolments (status);
CREATE INDEX idx_enrolments_compliance ON enrolments (due_date) WHERE status NOT IN ('completed', 'withdrawn');

-- =============================================================
-- SCORM RUNTIME DATA
-- =============================================================
CREATE TABLE scorm_runtime_data (
    runtime_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    enrolment_id    UUID NOT NULL REFERENCES enrolments(enrolment_id) ON DELETE CASCADE,
    attempt_number  INTEGER NOT NULL DEFAULT 1,
    sco_identifier  VARCHAR(255) NOT NULL,
    lesson_status   VARCHAR(30),                    -- 'passed', 'completed', 'failed', 'incomplete', 'browsed', 'not attempted'
    lesson_location VARCHAR(1000),
    score_raw       NUMERIC(10,4),
    score_min       NUMERIC(10,4),
    score_max       NUMERIC(10,4),
    score_scaled    NUMERIC(5,4),
    total_time      VARCHAR(50),                    -- SCORM timeinterval format
    session_time    VARCHAR(50),
    suspend_data    TEXT,                            -- up to 64000 chars for SCORM 2004
    entry           VARCHAR(20),
    exit_type       VARCHAR(20),
    credit          VARCHAR(10),
    interactions    TEXT,                            -- serialized interaction data
    objectives      TEXT,                            -- serialized objectives data
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (enrolment_id, attempt_number, sco_identifier)
);

CREATE INDEX idx_scorm_runtime_enrolment ON scorm_runtime_data (enrolment_id);

-- =============================================================
-- xAPI STATEMENTS (Learning Record Store)
-- =============================================================
CREATE TABLE xapi_statements (
    statement_id    UUID PRIMARY KEY,               -- xAPI requires client-set UUIDs
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    actor_user_id   UUID REFERENCES users(user_id),
    actor_mbox      VARCHAR(255),
    actor_account_name VARCHAR(255),
    verb_id         VARCHAR(500) NOT NULL,          -- IRI, e.g. http://adlnet.gov/expapi/verbs/completed
    verb_display    VARCHAR(255),
    object_type     VARCHAR(50) NOT NULL,           -- Activity, Agent, SubStatement, StatementRef
    object_id       VARCHAR(500) NOT NULL,          -- Activity IRI
    object_name     VARCHAR(500),
    result_score_scaled NUMERIC(5,4),
    result_score_raw    NUMERIC(10,4),
    result_score_min    NUMERIC(10,4),
    result_score_max    NUMERIC(10,4),
    result_success  BOOLEAN,
    result_completion BOOLEAN,
    result_duration VARCHAR(50),                    -- ISO 8601 duration
    result_response TEXT,
    context_registration UUID,
    context_instructor_id UUID REFERENCES users(user_id),
    context_platform VARCHAR(255),
    context_language VARCHAR(10),
    authority_agent VARCHAR(500),
    timestamp       TIMESTAMPTZ NOT NULL,
    stored          TIMESTAMPTZ NOT NULL DEFAULT now(),
    voided          BOOLEAN NOT NULL DEFAULT FALSE
);

CREATE INDEX idx_xapi_statements_tenant ON xapi_statements (tenant_id);
CREATE INDEX idx_xapi_statements_actor ON xapi_statements (actor_user_id);
CREATE INDEX idx_xapi_statements_verb ON xapi_statements (verb_id);
CREATE INDEX idx_xapi_statements_object ON xapi_statements (object_id);
CREATE INDEX idx_xapi_statements_timestamp ON xapi_statements (tenant_id, timestamp DESC);
CREATE INDEX idx_xapi_statements_registration ON xapi_statements (context_registration);

-- =============================================================
-- CERTIFICATIONS
-- =============================================================
CREATE TABLE certifications (
    certification_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id        UUID NOT NULL REFERENCES tenants(tenant_id),
    name             VARCHAR(500) NOT NULL,
    description      TEXT,
    issuing_body     VARCHAR(255),
    validity_days    INTEGER,                       -- NULL = never expires
    renewal_required BOOLEAN NOT NULL DEFAULT FALSE,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_certifications (
    user_cert_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    certification_id UUID NOT NULL REFERENCES certifications(certification_id),
    enrolment_id    UUID REFERENCES enrolments(enrolment_id),
    issued_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ,
    certificate_number VARCHAR(100),
    certificate_url TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'expired', 'revoked', 'renewed')),
    renewal_reminder_sent BOOLEAN NOT NULL DEFAULT FALSE
);

CREATE INDEX idx_user_certifications_user ON user_certifications (user_id);
CREATE INDEX idx_user_certifications_expiry ON user_certifications (expires_at) WHERE status = 'active';

-- =============================================================
-- COMPLIANCE TRAINING RULES
-- =============================================================
CREATE TABLE compliance_rules (
    rule_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    course_id       UUID NOT NULL REFERENCES courses(course_id),
    target_type     VARCHAR(30) NOT NULL CHECK (target_type IN ('all_users', 'organization', 'job_role', 'user_group')),
    target_id       UUID,                           -- NULL for 'all_users'
    frequency       VARCHAR(20) NOT NULL CHECK (frequency IN ('once', 'annual', 'biannual', 'quarterly', 'custom')),
    frequency_days  INTEGER,
    grace_period_days INTEGER NOT NULL DEFAULT 30,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_rules_tenant ON compliance_rules (tenant_id);
CREATE INDEX idx_compliance_rules_course ON compliance_rules (course_id);

-- =============================================================
-- MODULE PROGRESS TRACKING
-- =============================================================
CREATE TABLE module_progress (
    progress_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    enrolment_id    UUID NOT NULL REFERENCES enrolments(enrolment_id) ON DELETE CASCADE,
    module_id       UUID NOT NULL REFERENCES course_modules(module_id),
    status          VARCHAR(20) NOT NULL DEFAULT 'not_started'
        CHECK (status IN ('not_started', 'in_progress', 'completed', 'skipped')),
    completion_pct  NUMERIC(5,2) NOT NULL DEFAULT 0,
    time_spent_seconds INTEGER NOT NULL DEFAULT 0,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    last_accessed_at TIMESTAMPTZ,
    UNIQUE (enrolment_id, module_id)
);

CREATE INDEX idx_module_progress_enrolment ON module_progress (enrolment_id);
```

### Domain 5: Analytics & ROI

```sql
-- =============================================================
-- LEARNING EVENTS (aggregated for analytics)
-- =============================================================
CREATE TABLE learning_events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    user_id         UUID NOT NULL REFERENCES users(user_id),
    event_type      VARCHAR(50) NOT NULL CHECK (event_type IN (
        'course_started', 'course_completed', 'course_failed',
        'assessment_passed', 'assessment_failed',
        'skill_level_changed', 'certification_earned', 'certification_expired',
        'pathway_started', 'pathway_completed',
        'content_rated', 'nudge_sent', 'nudge_clicked'
    )),
    course_id       UUID REFERENCES courses(course_id),
    skill_id        UUID REFERENCES skills(skill_id),
    pathway_id      UUID REFERENCES learning_pathways(pathway_id),
    event_data      JSONB,                          -- additional event-specific payload
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_learning_events_tenant_time ON learning_events (tenant_id, occurred_at DESC);
CREATE INDEX idx_learning_events_user ON learning_events (user_id, occurred_at DESC);
CREATE INDEX idx_learning_events_type ON learning_events (event_type);

-- =============================================================
-- BUSINESS KPI SNAPSHOTS (for ROI correlation)
-- =============================================================
CREATE TABLE business_kpi_definitions (
    kpi_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    kpi_category    VARCHAR(50) NOT NULL CHECK (kpi_category IN ('sales', 'retention', 'promotion', 'quality', 'customer_satisfaction', 'productivity', 'safety', 'custom')),
    unit            VARCHAR(50),                    -- e.g., 'percentage', 'currency', 'count'
    higher_is_better BOOLEAN NOT NULL DEFAULT TRUE,
    data_source     VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE business_kpi_values (
    value_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    kpi_id          UUID NOT NULL REFERENCES business_kpi_definitions(kpi_id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(user_id),
    org_id          UUID REFERENCES organizations(org_id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    value           NUMERIC(15,4) NOT NULL,
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
    report_type     VARCHAR(30) NOT NULL CHECK (report_type IN ('individual', 'team', 'department', 'organization', 'program')),
    scope_type      VARCHAR(30) NOT NULL,           -- matches report_type for filtering
    scope_id        UUID,
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    generated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    generated_by    UUID REFERENCES users(user_id),
    total_learning_hours NUMERIC(10,2),
    total_completions INTEGER,
    total_certifications_earned INTEGER,
    kpi_impact_summary JSONB,                       -- aggregated KPI deltas
    status          VARCHAR(20) NOT NULL DEFAULT 'generated' CHECK (status IN ('generating', 'generated', 'published', 'archived'))
);

CREATE TABLE roi_report_correlations (
    correlation_id  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_id       UUID NOT NULL REFERENCES roi_reports(report_id) ON DELETE CASCADE,
    kpi_id          UUID NOT NULL REFERENCES business_kpi_definitions(kpi_id),
    course_id       UUID REFERENCES courses(course_id),
    pathway_id      UUID REFERENCES learning_pathways(pathway_id),
    learner_cohort_size INTEGER,
    control_cohort_size INTEGER,
    kpi_before      NUMERIC(15,4),
    kpi_after       NUMERIC(15,4),
    delta           NUMERIC(15,4),
    delta_pct       NUMERIC(8,4),
    confidence_level NUMERIC(5,4),
    is_significant  BOOLEAN,
    attribution_method VARCHAR(50)                  -- e.g., 'pre_post', 'control_group', 'regression'
);
```

### Domain 6: Integration & Audit

```sql
-- =============================================================
-- INTEGRATION SYNC LOG
-- =============================================================
CREATE TABLE integration_sync_log (
    sync_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    integration_type VARCHAR(30) NOT NULL CHECK (integration_type IN ('scim', 'hris', 'content_provider', 'lti', 'slack', 'teams', 'kpi_import')),
    direction       VARCHAR(10) NOT NULL CHECK (direction IN ('inbound', 'outbound')),
    status          VARCHAR(20) NOT NULL CHECK (status IN ('started', 'completed', 'failed', 'partial')),
    records_processed INTEGER NOT NULL DEFAULT 0,
    records_created INTEGER NOT NULL DEFAULT 0,
    records_updated INTEGER NOT NULL DEFAULT 0,
    records_failed  INTEGER NOT NULL DEFAULT 0,
    error_details   TEXT,
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_sync_log_tenant ON integration_sync_log (tenant_id, started_at DESC);

-- =============================================================
-- NOTIFICATIONS
-- =============================================================
CREATE TABLE notifications (
    notification_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    user_id         UUID NOT NULL REFERENCES users(user_id),
    channel         VARCHAR(20) NOT NULL CHECK (channel IN ('in_app', 'email', 'slack', 'teams', 'push')),
    notification_type VARCHAR(50) NOT NULL,
    title           VARCHAR(500),
    body            TEXT,
    action_url      TEXT,
    is_read         BOOLEAN NOT NULL DEFAULT FALSE,
    sent_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    read_at         TIMESTAMPTZ,
    delivered       BOOLEAN NOT NULL DEFAULT FALSE
);

CREATE INDEX idx_notifications_user ON notifications (user_id, sent_at DESC);
CREATE INDEX idx_notifications_unread ON notifications (user_id) WHERE is_read = FALSE;

-- =============================================================
-- AUDIT TRAIL
-- =============================================================
CREATE TABLE audit_log (
    log_id          BIGSERIAL PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    user_id         UUID,
    action          VARCHAR(50) NOT NULL,           -- e.g., 'create', 'update', 'delete', 'login', 'export'
    resource_type   VARCHAR(100) NOT NULL,          -- e.g., 'course', 'enrolment', 'user', 'compliance_rule'
    resource_id     UUID,
    old_values      JSONB,
    new_values      JSONB,
    ip_address      INET,
    user_agent      TEXT,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_tenant_time ON audit_log (tenant_id, occurred_at DESC);
CREATE INDEX idx_audit_log_user ON audit_log (user_id, occurred_at DESC);
CREATE INDEX idx_audit_log_resource ON audit_log (resource_type, resource_id);

-- Partition audit log by month for performance
-- In production, use pg_partman or native PARTITION BY RANGE (occurred_at)
```

### Key Views

```sql
-- =============================================================
-- MATERIALIZED VIEW: Team Skill Heatmap
-- =============================================================
CREATE MATERIALIZED VIEW mv_team_skill_heatmap AS
SELECT
    o.org_id,
    o.name AS team_name,
    s.skill_id,
    s.name AS skill_name,
    s.skill_type,
    AVG(us.current_level) AS avg_level,
    MIN(us.current_level) AS min_level,
    MAX(us.current_level) AS max_level,
    COUNT(DISTINCT us.user_id) AS assessed_count,
    COUNT(DISTINCT u.user_id) AS team_size
FROM organizations o
JOIN users u ON u.org_id = o.org_id AND u.status = 'active'
LEFT JOIN user_skills us ON us.user_id = u.user_id
LEFT JOIN skills s ON s.skill_id = us.skill_id
GROUP BY o.org_id, o.name, s.skill_id, s.name, s.skill_type;

CREATE UNIQUE INDEX idx_mv_team_skill_heatmap ON mv_team_skill_heatmap (org_id, skill_id);

-- =============================================================
-- VIEW: Compliance Status Dashboard
-- =============================================================
CREATE VIEW vw_compliance_status AS
SELECT
    cr.rule_id,
    cr.name AS rule_name,
    cr.tenant_id,
    u.user_id,
    u.display_name,
    o.name AS department,
    c.title AS course_title,
    e.status AS enrolment_status,
    e.completed_at,
    cr.frequency,
    CASE
        WHEN e.enrolment_id IS NULL THEN 'not_enrolled'
        WHEN e.status = 'completed' AND (cr.frequency = 'once' OR
            e.completed_at + (cr.frequency_days || ' days')::interval > now()) THEN 'compliant'
        WHEN e.due_date < now() AND e.status != 'completed' THEN 'overdue'
        ELSE 'in_progress'
    END AS compliance_status,
    e.due_date,
    GREATEST(0, EXTRACT(DAY FROM e.due_date - now()))::INTEGER AS days_remaining
FROM compliance_rules cr
CROSS JOIN users u
JOIN courses c ON c.course_id = cr.course_id
LEFT JOIN organizations o ON o.org_id = u.org_id
LEFT JOIN enrolments e ON e.user_id = u.user_id AND e.course_id = cr.course_id
WHERE cr.is_active = TRUE AND u.status = 'active'
    AND u.tenant_id = cr.tenant_id;

-- =============================================================
-- MATERIALIZED VIEW: Learning Activity Summary
-- =============================================================
CREATE MATERIALIZED VIEW mv_learning_activity_summary AS
SELECT
    e.user_id,
    DATE_TRUNC('month', e.completed_at) AS month,
    COUNT(*) FILTER (WHERE e.status = 'completed') AS courses_completed,
    SUM(e.time_spent_seconds) / 3600.0 AS total_hours,
    AVG(e.score) FILTER (WHERE e.score IS NOT NULL) AS avg_score,
    COUNT(*) FILTER (WHERE c.is_compliance = TRUE AND e.status = 'completed') AS compliance_completed
FROM enrolments e
JOIN courses c ON c.course_id = e.course_id
WHERE e.completed_at IS NOT NULL
GROUP BY e.user_id, DATE_TRUNC('month', e.completed_at);

CREATE UNIQUE INDEX idx_mv_learning_activity ON mv_learning_activity_summary (user_id, month);
```

---

## Pros and Cons

### Pros

1. **Data integrity guarantee**: Foreign keys and constraints ensure referential integrity across all domains — critical for compliance audit trails where you must prove a specific user completed a specific course at a specific time.

2. **Mature tooling**: PostgreSQL has decades of tooling for backup, replication, monitoring, and migration. Every ORM and application framework supports it natively.

3. **Standards compliance**: The normalized schema maps directly to xAPI statement structure, SCORM runtime data elements, SCIM user attributes, and LTI deployment concepts, making standards integration straightforward.

4. **Query flexibility**: Complex analytical queries (skill gap calculations across departments, compliance status rollups, ROI correlations) are natural in SQL with JOINs and window functions.

5. **ACID transactions**: Enrolment workflows, compliance rule evaluation, and certification issuance can run in single transactions ensuring consistency.

6. **Clear audit trail**: Every entity has timestamps and the audit_log table captures all mutations for regulatory requirements.

### Cons

1. **Schema rigidity**: Adding new xAPI extensions, custom assessment types, or tenant-specific fields requires DDL migrations. The skills taxonomy structure is locked to the table definition.

2. **xAPI storage overhead**: Storing xAPI statements in a relational table loses the natural JSON document structure. Complex context and extension data must be either flattened (losing fidelity) or stored as TEXT/JSONB (losing queryability without GIN indexes).

3. **Performance at scale**: The xapi_statements table will grow rapidly (potentially millions of rows per month). Without partitioning, queries degrade. The fully normalized structure requires many JOINs for common read paths.

4. **Hierarchical data friction**: Skill categories, organization trees, and course module hierarchies require recursive CTEs which are less efficient than purpose-built tree structures.

5. **Skill relationship queries are expensive**: Finding all skills related to a given skill through multiple hops (prerequisite chains, skill clusters) requires recursive queries that perform poorly compared to graph databases.

6. **Multi-tenancy complexity**: Row-level tenant_id filtering on every query adds complexity and risk of data leakage if forgotten. Row-level security policies help but add overhead.

---

## Migration and Scaling Considerations

### Migration Strategy

1. **Schema versioning**: Use Flyway or golang-migrate with numbered migration files. Each migration is idempotent and includes both up and down scripts.

2. **Zero-downtime migrations**: For large tables (xapi_statements, audit_log), use `CREATE INDEX CONCURRENTLY` and `ALTER TABLE ... ADD COLUMN` with defaults to avoid locks.

3. **Data import**: ESCO and O*NET data imports should use `COPY` command for bulk loading into skill_frameworks, skill_categories, and skills tables. ESCO provides ~13,000 skills and ~3,000 occupations.

### Scaling Strategy

1. **Read replicas**: Deploy 1-2 streaming replicas for analytics queries, reports, and the materialized view refresh workload.

2. **Table partitioning**: Partition xapi_statements and audit_log by month using PostgreSQL native partitioning:
   ```sql
   CREATE TABLE xapi_statements (
       ...
   ) PARTITION BY RANGE (stored);
   ```

3. **Connection pooling**: PgBouncer in transaction mode to handle thousands of concurrent learner sessions.

4. **Materialized view refresh**: Schedule `REFRESH MATERIALIZED VIEW CONCURRENTLY` during off-peak hours for dashboard views.

5. **Archival**: Move completed enrolments and old xAPI statements older than the retention policy to archive tables or cold storage.

6. **Estimated capacity**: For a 50,000-user organization generating ~10 xAPI statements per user per day, expect ~500K rows/day in xapi_statements (~180M/year). Partitioning and archival are essential beyond year one.

### Multi-Tenant Isolation

The schema uses a shared-schema, shared-database approach with `tenant_id` on every table. For larger deployments, consider:
- PostgreSQL Row Level Security (RLS) policies to enforce tenant isolation at the database level
- Schema-per-tenant for high-isolation requirements
- Separate database instances for regulated industries requiring physical data separation
