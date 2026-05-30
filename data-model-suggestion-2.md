# Data Model Suggestion 2: Event-Sourced / CQRS Model

## Overview

This model treats every learning activity, skill change, enrolment decision, and compliance action as an immutable event. Instead of storing current state in mutable rows, the system persists an append-only stream of domain events and derives current state by replaying or projecting those events into purpose-built read models.

For a corporate L&D platform, event sourcing is particularly compelling because:

- **Learning is inherently event-driven**: A learner started a module, answered a question, paused, resumed, completed, scored — these are naturally sequential events.
- **Audit trails are free**: Compliance and regulatory requirements demand a complete, tamper-proof history of who did what and when. In event sourcing, the audit trail IS the data.
- **xAPI is already event-shaped**: xAPI statements (Actor-Verb-Object-Result-Context) are essentially domain events. An event-sourced L&D platform can natively store xAPI as first-class events without translation.
- **Skill evolution tracking**: Understanding how skills change over time (not just current state) enables AI-driven trend analysis and prediction.

The architecture uses CQRS (Command Query Responsibility Segregation) to separate write operations (event appends) from read operations (projections/views), allowing each side to scale independently.

---

## Technology Recommendations

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Event Store | EventStoreDB or PostgreSQL with event tables | EventStoreDB is purpose-built; PostgreSQL provides familiarity and transactional guarantees |
| Message Broker | Apache Kafka or NATS JetStream | Durable event streaming for projections and integrations |
| Read Model Store | PostgreSQL | Projections need flexible querying and JOIN support |
| Cache Layer | Redis | Fast read model caching for dashboards and real-time views |
| Search Index | Elasticsearch / OpenSearch | Full-text search over courses, skills, and xAPI statements |
| Application Framework | Node.js/TypeScript or .NET with MediatR | Strong CQRS library ecosystems |
| Snapshot Store | PostgreSQL or S3 | Periodic aggregate snapshots to speed replay |

---

## Event Store Schema

The event store is the single source of truth. All state changes flow through it.

### Core Event Store Tables (PostgreSQL Implementation)

```sql
-- =============================================================
-- EVENT STORE: The single source of truth
-- =============================================================
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       VARCHAR(500) NOT NULL,          -- e.g., 'user-{uuid}', 'enrolment-{uuid}'
    stream_type     VARCHAR(100) NOT NULL,           -- aggregate type: 'User', 'Enrolment', 'Course', etc.
    event_type      VARCHAR(200) NOT NULL,           -- e.g., 'LearnerEnrolled', 'ModuleCompleted'
    event_version   INTEGER NOT NULL,               -- position within the stream (optimistic concurrency)
    tenant_id       UUID NOT NULL,
    data            JSONB NOT NULL,                 -- event payload
    metadata        JSONB NOT NULL DEFAULT '{}',    -- correlation IDs, causation IDs, user agent, IP
    occurred_at     TIMESTAMPTZ NOT NULL,           -- when the event happened in the domain
    stored_at       TIMESTAMPTZ NOT NULL DEFAULT now(), -- when we persisted it
    UNIQUE (stream_id, event_version)               -- optimistic concurrency control
) PARTITION BY RANGE (stored_at);

-- Create monthly partitions (automate with pg_partman in production)
CREATE TABLE event_store_2026_01 PARTITION OF event_store
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE event_store_2026_02 PARTITION OF event_store
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... continue for each month

CREATE INDEX idx_event_store_stream ON event_store (stream_id, event_version);
CREATE INDEX idx_event_store_type ON event_store (event_type, stored_at);
CREATE INDEX idx_event_store_tenant ON event_store (tenant_id, stored_at);
CREATE INDEX idx_event_store_correlation ON event_store ((metadata->>'correlation_id'))
    WHERE metadata->>'correlation_id' IS NOT NULL;

-- =============================================================
-- STREAM METADATA (current version tracking for optimistic concurrency)
-- =============================================================
CREATE TABLE stream_metadata (
    stream_id       VARCHAR(500) PRIMARY KEY,
    stream_type     VARCHAR(100) NOT NULL,
    tenant_id       UUID NOT NULL,
    current_version INTEGER NOT NULL DEFAULT 0,
    snapshot_version INTEGER,
    snapshot_data   JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =============================================================
-- PROJECTION CHECKPOINTS (track what each projection has processed)
-- =============================================================
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(200) PRIMARY KEY,
    last_event_id   UUID,
    last_position   BIGINT NOT NULL DEFAULT 0,
    last_processed_at TIMESTAMPTZ,
    status          VARCHAR(20) NOT NULL DEFAULT 'running'
        CHECK (status IN ('running', 'paused', 'rebuilding', 'failed')),
    error_message   TEXT,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =============================================================
-- DEAD LETTER QUEUE (failed event processing)
-- =============================================================
CREATE TABLE dead_letter_events (
    dead_letter_id  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL,
    projection_name VARCHAR(200) NOT NULL,
    error_message   TEXT NOT NULL,
    retry_count     INTEGER NOT NULL DEFAULT 0,
    max_retries     INTEGER NOT NULL DEFAULT 5,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_retry_at   TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ
);
```

---

## Domain Event Types

### User Aggregate Events

```sql
-- Example event payloads stored in event_store.data column

-- Stream: user-{userId}
-- Events:
```

```json
// UserCreated
{
    "event_type": "UserCreated",
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "tenant_id": "...",
    "username": "jsmith",
    "email": "jane.smith@company.com",
    "first_name": "Jane",
    "last_name": "Smith",
    "job_title": "Senior Engineer",
    "org_id": "...",
    "manager_id": "...",
    "source": "scim_sync"
}

// UserProfileUpdated
{
    "event_type": "UserProfileUpdated",
    "user_id": "...",
    "changes": {
        "job_title": { "from": "Senior Engineer", "to": "Staff Engineer" },
        "org_id": { "from": "...", "to": "..." }
    }
}

// UserDeactivated
{
    "event_type": "UserDeactivated",
    "user_id": "...",
    "reason": "employment_ended",
    "deactivated_by": "..."
}
```

### Enrolment Aggregate Events

```json
// LearnerEnrolled
{
    "event_type": "LearnerEnrolled",
    "enrolment_id": "...",
    "user_id": "...",
    "course_id": "...",
    "enrolment_type": "compliance_required",
    "assigned_by": "...",
    "due_date": "2026-06-30",
    "compliance_rule_id": "..."
}

// CourseStarted
{
    "event_type": "CourseStarted",
    "enrolment_id": "...",
    "user_id": "...",
    "course_id": "...",
    "started_at": "2026-05-20T09:30:00Z"
}

// ModuleStarted
{
    "event_type": "ModuleStarted",
    "enrolment_id": "...",
    "module_id": "...",
    "started_at": "2026-05-20T09:31:00Z"
}

// ModuleCompleted
{
    "event_type": "ModuleCompleted",
    "enrolment_id": "...",
    "module_id": "...",
    "time_spent_seconds": 1845,
    "completion_pct": 100,
    "completed_at": "2026-05-20T10:02:00Z"
}

// AssessmentSubmitted
{
    "event_type": "AssessmentSubmitted",
    "enrolment_id": "...",
    "assessment_id": "...",
    "attempt_number": 1,
    "answers": [
        {"question_id": "...", "selected_option_id": "...", "is_correct": true},
        {"question_id": "...", "response_text": "...", "score": 8.5}
    ],
    "total_score": 87.5,
    "passed": true,
    "time_spent_seconds": 900
}

// CourseCompleted
{
    "event_type": "CourseCompleted",
    "enrolment_id": "...",
    "user_id": "...",
    "course_id": "...",
    "final_score": 92.3,
    "total_time_seconds": 14400,
    "completed_at": "2026-05-22T16:00:00Z"
}

// CourseFailed
{
    "event_type": "CourseFailed",
    "enrolment_id": "...",
    "user_id": "...",
    "course_id": "...",
    "final_score": 55.0,
    "attempts_used": 3,
    "max_attempts": 3,
    "failed_at": "2026-05-25T11:00:00Z"
}
```

### SCORM / xAPI Runtime Events

```json
// ScormSessionInitialized
{
    "event_type": "ScormSessionInitialized",
    "enrolment_id": "...",
    "sco_identifier": "module_1_intro",
    "scorm_version": "2004_4th",
    "attempt_number": 1,
    "entry": "ab-initio"
}

// ScormDataCommitted
{
    "event_type": "ScormDataCommitted",
    "enrolment_id": "...",
    "sco_identifier": "module_1_intro",
    "attempt_number": 1,
    "cmi_data": {
        "lesson_location": "page_7",
        "lesson_status": "incomplete",
        "score_raw": null,
        "suspend_data": "base64encodeddata...",
        "session_time": "PT15M32S"
    }
}

// ScormSessionTerminated
{
    "event_type": "ScormSessionTerminated",
    "enrolment_id": "...",
    "sco_identifier": "module_1_intro",
    "attempt_number": 1,
    "final_status": "completed",
    "score_raw": 85,
    "score_scaled": 0.85,
    "total_time": "PT45M12S",
    "exit_type": "normal"
}

// XApiStatementReceived (native xAPI ingestion)
{
    "event_type": "XApiStatementReceived",
    "statement_id": "...",
    "actor": {
        "mbox": "mailto:jane.smith@company.com",
        "name": "Jane Smith"
    },
    "verb": {
        "id": "http://adlnet.gov/expapi/verbs/completed",
        "display": {"en-US": "completed"}
    },
    "object": {
        "id": "https://lms.company.com/courses/leadership-101",
        "definition": {
            "name": {"en-US": "Leadership Fundamentals"},
            "type": "http://adlnet.gov/expapi/activities/course"
        }
    },
    "result": {
        "score": {"scaled": 0.92, "raw": 92, "min": 0, "max": 100},
        "success": true,
        "completion": true,
        "duration": "PT4H30M"
    },
    "context": {
        "registration": "...",
        "platform": "Corporate LMS",
        "language": "en-US"
    },
    "timestamp": "2026-05-20T16:00:00Z"
}
```

### Skills Aggregate Events

```json
// SkillAssessed
{
    "event_type": "SkillAssessed",
    "user_id": "...",
    "skill_id": "...",
    "previous_level": 3,
    "new_level": 5,
    "assessment_method": "certification",
    "evidence_url": "https://...",
    "assessed_by": "...",
    "assessed_at": "2026-05-20T10:00:00Z"
}

// SkillGapAnalysisGenerated
{
    "event_type": "SkillGapAnalysisGenerated",
    "analysis_id": "...",
    "target_type": "team",
    "target_id": "...",
    "job_role_id": "...",
    "gaps": [
        {"skill_id": "...", "skill_name": "Kubernetes", "required": 7, "current": 3, "gap": 4, "priority": "critical"},
        {"skill_id": "...", "skill_name": "Python", "required": 6, "current": 5, "gap": 1, "priority": "low"}
    ],
    "overall_readiness_pct": 72.5,
    "generated_by": "ai"
}

// SkillInferredFromWorkArtifact
{
    "event_type": "SkillInferredFromWorkArtifact",
    "user_id": "...",
    "skill_id": "...",
    "inferred_level": 6,
    "confidence": 0.85,
    "artifact_source": "github_commits",
    "artifact_reference": "https://github.com/org/repo/pulls?q=author:jsmith",
    "inference_model_version": "v2.3"
}
```

### Pathway & Certification Events

```json
// LearningPathwayAssigned
{
    "event_type": "LearningPathwayAssigned",
    "pathway_id": "...",
    "user_id": "...",
    "assigned_by": "ai_recommendation",
    "target_role_id": "...",
    "steps": [
        {"step_id": "...", "course_id": "...", "sort_order": 1, "is_required": true},
        {"step_id": "...", "course_id": "...", "sort_order": 2, "is_required": true}
    ],
    "estimated_duration_days": 30
}

// PathwayStepCompleted
{
    "event_type": "PathwayStepCompleted",
    "pathway_id": "...",
    "user_id": "...",
    "step_id": "...",
    "course_id": "...",
    "completed_at": "2026-05-22T16:00:00Z"
}

// CertificationEarned
{
    "event_type": "CertificationEarned",
    "user_cert_id": "...",
    "user_id": "...",
    "certification_id": "...",
    "enrolment_id": "...",
    "certificate_number": "CERT-2026-00451",
    "issued_at": "2026-05-22T16:30:00Z",
    "expires_at": "2027-05-22T16:30:00Z"
}

// CertificationExpired
{
    "event_type": "CertificationExpired",
    "user_cert_id": "...",
    "user_id": "...",
    "certification_id": "...",
    "expired_at": "2027-05-22T16:30:00Z"
}
```

### AI & Analytics Events

```json
// ContentQualityScoreCalculated
{
    "event_type": "ContentQualityScoreCalculated",
    "course_id": "...",
    "quality_score": 0.78,
    "factors": {
        "freshness": 0.9,
        "learner_ratings_avg": 4.2,
        "completion_rate": 0.73,
        "source_credibility": 0.85,
        "content_age_months": 6
    },
    "calculated_at": "2026-05-25T02:00:00Z"
}

// LearningImpactCorrelationDetected
{
    "event_type": "LearningImpactCorrelationDetected",
    "correlation_id": "...",
    "course_id": "...",
    "kpi_id": "...",
    "kpi_name": "sales_conversion_rate",
    "learner_cohort_size": 45,
    "control_cohort_size": 52,
    "kpi_before": 12.3,
    "kpi_after": 15.7,
    "delta_pct": 27.6,
    "confidence_level": 0.94,
    "is_significant": true,
    "attribution_method": "control_group"
}

// PathwayEffectivenessEvaluated
{
    "event_type": "PathwayEffectivenessEvaluated",
    "pathway_id": "...",
    "evaluation_period_start": "2026-01-01",
    "evaluation_period_end": "2026-03-31",
    "learners_enrolled": 120,
    "learners_completed": 87,
    "avg_completion_days": 22,
    "target_kpis_improved": true,
    "recommendation": "keep_active"
}

// NudgeSent
{
    "event_type": "NudgeSent",
    "notification_id": "...",
    "user_id": "...",
    "channel": "slack",
    "nudge_type": "learning_reminder",
    "course_id": "...",
    "message": "You're 60% through Leadership Fundamentals. 15 minutes today would complete Module 4!",
    "sent_at": "2026-05-25T09:00:00Z"
}

// NudgeInteracted
{
    "event_type": "NudgeInteracted",
    "notification_id": "...",
    "user_id": "...",
    "interaction_type": "clicked",
    "interacted_at": "2026-05-25T09:12:00Z"
}
```

### Compliance Events

```json
// ComplianceRuleCreated
{
    "event_type": "ComplianceRuleCreated",
    "rule_id": "...",
    "tenant_id": "...",
    "name": "Annual Security Awareness Training",
    "course_id": "...",
    "target_type": "all_users",
    "frequency": "annual",
    "grace_period_days": 30,
    "created_by": "..."
}

// ComplianceAssignmentTriggered
{
    "event_type": "ComplianceAssignmentTriggered",
    "rule_id": "...",
    "user_ids": ["...", "...", "..."],
    "course_id": "...",
    "due_date": "2026-07-01",
    "triggered_reason": "annual_cycle"
}

// ComplianceDeadlineMissed
{
    "event_type": "ComplianceDeadlineMissed",
    "rule_id": "...",
    "user_id": "...",
    "course_id": "...",
    "due_date": "2026-06-01",
    "escalation_level": 1,
    "manager_notified": true
}
```

---

## Read Model Projections

Projections subscribe to the event stream and build optimized read models. Each projection runs independently and can be rebuilt from scratch by replaying events.

### Projection: Current Enrolment State

```sql
-- This table is DERIVED (rebuilt from events). Never written to directly.
CREATE TABLE rm_enrolments (
    enrolment_id    UUID PRIMARY KEY,
    user_id         UUID NOT NULL,
    course_id       UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    pathway_id      UUID,
    enrolment_type  VARCHAR(30) NOT NULL,
    status          VARCHAR(20) NOT NULL,
    assigned_by     UUID,
    due_date        DATE,
    enrolled_at     TIMESTAMPTZ NOT NULL,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    score           NUMERIC(5,2),
    attempt_count   INTEGER NOT NULL DEFAULT 0,
    time_spent_seconds INTEGER NOT NULL DEFAULT 0,
    completion_pct  NUMERIC(5,2) NOT NULL DEFAULT 0,
    last_activity_at TIMESTAMPTZ,
    last_event_version INTEGER NOT NULL,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_enrolments_user ON rm_enrolments (user_id);
CREATE INDEX idx_rm_enrolments_course ON rm_enrolments (course_id);
CREATE INDEX idx_rm_enrolments_status ON rm_enrolments (tenant_id, status);
CREATE INDEX idx_rm_enrolments_compliance ON rm_enrolments (due_date)
    WHERE status NOT IN ('completed', 'withdrawn');
```

### Projection: User Skill Profile

```sql
CREATE TABLE rm_user_skills (
    user_id         UUID NOT NULL,
    skill_id        UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    skill_name      VARCHAR(500) NOT NULL,
    skill_type      VARCHAR(30) NOT NULL,
    current_level   INTEGER NOT NULL,
    target_level    INTEGER,
    assessment_method VARCHAR(30) NOT NULL,
    last_assessed_at TIMESTAMPTZ NOT NULL,
    level_history   JSONB NOT NULL DEFAULT '[]',    -- [{level: 3, date: "...", method: "..."}, ...]
    trend           VARCHAR(10),                    -- 'improving', 'stable', 'declining'
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, skill_id)
);

CREATE INDEX idx_rm_user_skills_tenant ON rm_user_skills (tenant_id);
CREATE INDEX idx_rm_user_skills_skill ON rm_user_skills (skill_id);
```

### Projection: Compliance Dashboard

```sql
CREATE TABLE rm_compliance_status (
    tenant_id       UUID NOT NULL,
    rule_id         UUID NOT NULL,
    user_id         UUID NOT NULL,
    rule_name       VARCHAR(255) NOT NULL,
    course_id       UUID NOT NULL,
    course_title    VARCHAR(500) NOT NULL,
    user_display_name VARCHAR(255),
    department      VARCHAR(255),
    compliance_status VARCHAR(20) NOT NULL,          -- 'compliant', 'in_progress', 'overdue', 'not_enrolled'
    due_date        DATE,
    completed_at    TIMESTAMPTZ,
    days_overdue    INTEGER,
    escalation_level INTEGER NOT NULL DEFAULT 0,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (rule_id, user_id)
);

CREATE INDEX idx_rm_compliance_tenant ON rm_compliance_status (tenant_id);
CREATE INDEX idx_rm_compliance_overdue ON rm_compliance_status (tenant_id, compliance_status)
    WHERE compliance_status = 'overdue';
```

### Projection: Team Skill Heatmap

```sql
CREATE TABLE rm_team_skill_heatmap (
    org_id          UUID NOT NULL,
    skill_id        UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    team_name       VARCHAR(255) NOT NULL,
    skill_name      VARCHAR(500) NOT NULL,
    skill_type      VARCHAR(30) NOT NULL,
    avg_level       NUMERIC(4,2) NOT NULL,
    min_level       INTEGER NOT NULL,
    max_level       INTEGER NOT NULL,
    assessed_count  INTEGER NOT NULL,
    team_size       INTEGER NOT NULL,
    coverage_pct    NUMERIC(5,2) NOT NULL,           -- assessed_count / team_size * 100
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (org_id, skill_id)
);

CREATE INDEX idx_rm_heatmap_tenant ON rm_team_skill_heatmap (tenant_id);
```

### Projection: Learning Activity Timeline

```sql
CREATE TABLE rm_learning_timeline (
    event_id        UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    user_id         UUID NOT NULL,
    event_type      VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,           -- human-readable summary
    description     TEXT,
    course_id       UUID,
    course_title    VARCHAR(500),
    skill_id        UUID,
    skill_name      VARCHAR(500),
    score           NUMERIC(5,2),
    occurred_at     TIMESTAMPTZ NOT NULL,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_timeline_user ON rm_learning_timeline (user_id, occurred_at DESC);
CREATE INDEX idx_rm_timeline_tenant ON rm_learning_timeline (tenant_id, occurred_at DESC);
```

### Projection: ROI Analytics

```sql
CREATE TABLE rm_roi_metrics (
    tenant_id       UUID NOT NULL,
    period_month    DATE NOT NULL,                  -- first of month
    total_users     INTEGER NOT NULL,
    active_learners INTEGER NOT NULL,
    courses_completed INTEGER NOT NULL,
    total_learning_hours NUMERIC(10,2) NOT NULL,
    certifications_earned INTEGER NOT NULL,
    compliance_completion_rate NUMERIC(5,2),
    avg_score       NUMERIC(5,2),
    kpi_correlations JSONB,                         -- [{kpi: "...", delta_pct: 12.3, significant: true}, ...]
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, period_month)
);
```

### Projection: xAPI Statement Store (LRS Read Model)

```sql
-- Optimized read model for xAPI statement queries
CREATE TABLE rm_xapi_statements (
    statement_id    UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    actor_user_id   UUID,
    actor_identifier VARCHAR(500) NOT NULL,          -- mbox, account, or openid
    verb_id         VARCHAR(500) NOT NULL,
    verb_display    VARCHAR(255),
    object_id       VARCHAR(500) NOT NULL,
    object_type     VARCHAR(50) NOT NULL,
    object_name     VARCHAR(500),
    result_score_scaled NUMERIC(5,4),
    result_success  BOOLEAN,
    result_completion BOOLEAN,
    result_duration VARCHAR(50),
    context_registration UUID,
    context_platform VARCHAR(255),
    timestamp       TIMESTAMPTZ NOT NULL,
    stored          TIMESTAMPTZ NOT NULL,
    full_statement  JSONB NOT NULL,                 -- complete xAPI statement for conformance
    voided          BOOLEAN NOT NULL DEFAULT FALSE
);

CREATE INDEX idx_rm_xapi_tenant ON rm_xapi_statements (tenant_id, stored DESC);
CREATE INDEX idx_rm_xapi_actor ON rm_xapi_statements (actor_user_id, stored DESC);
CREATE INDEX idx_rm_xapi_verb ON rm_xapi_statements (verb_id);
CREATE INDEX idx_rm_xapi_object ON rm_xapi_statements (object_id);
CREATE INDEX idx_rm_xapi_registration ON rm_xapi_statements (context_registration);
CREATE INDEX idx_rm_xapi_full ON rm_xapi_statements USING gin (full_statement);
```

### Projection: SCORM Runtime State

```sql
-- Current SCORM runtime state derived from ScormDataCommitted and ScormSessionTerminated events
CREATE TABLE rm_scorm_runtime (
    enrolment_id    UUID NOT NULL,
    sco_identifier  VARCHAR(255) NOT NULL,
    attempt_number  INTEGER NOT NULL,
    tenant_id       UUID NOT NULL,
    lesson_status   VARCHAR(30),
    lesson_location VARCHAR(1000),
    score_raw       NUMERIC(10,4),
    score_min       NUMERIC(10,4),
    score_max       NUMERIC(10,4),
    score_scaled    NUMERIC(5,4),
    total_time      VARCHAR(50),
    suspend_data    TEXT,
    entry           VARCHAR(20),
    exit_type       VARCHAR(20),
    commit_count    INTEGER NOT NULL DEFAULT 0,       -- how many times data was committed
    last_commit_at  TIMESTAMPTZ,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (enrolment_id, sco_identifier, attempt_number)
);
```

---

## Command Handlers

Commands are the write side of CQRS. Each command validates business rules, produces events, and appends them to the event store.

### Example Command: EnrolLearner

```
Command: EnrolLearner
  Input: { userId, courseId, enrolmentType, assignedBy?, dueDate?, complianceRuleId? }
  
  Validation:
    1. User exists and is active
    2. Course exists and is published
    3. User is not already enrolled in this course (active enrolment)
    4. If compliance: verify rule applies to this user
    5. If pathway: verify prerequisite steps completed
  
  Events produced:
    -> LearnerEnrolled { enrolmentId, userId, courseId, enrolmentType, assignedBy, dueDate }
    -> If compliance: ComplianceAssignmentTriggered { ruleId, userId, courseId, dueDate }
```

### Example Command: SubmitAssessment

```
Command: SubmitAssessment
  Input: { enrolmentId, assessmentId, answers[] }
  
  Validation:
    1. Enrolment is active (in_progress or enrolled)
    2. Assessment belongs to the enrolled course
    3. Attempt count < max_attempts
    4. If proctored: verify proctoring session valid
  
  Processing:
    1. Grade each answer against correct options
    2. Calculate total score
    3. Determine pass/fail against passing_score
  
  Events produced:
    -> AssessmentSubmitted { enrolmentId, assessmentId, attemptNumber, answers, totalScore, passed }
    -> If passed and all modules complete:
       -> CourseCompleted { enrolmentId, userId, courseId, finalScore }
       -> SkillAssessed { userId, skillId, newLevel, method: 'assessment' } (for each mapped skill)
       -> If certification course: CertificationEarned { ... }
    -> If failed and attempts exhausted:
       -> CourseFailed { enrolmentId, userId, courseId, finalScore }
```

### Example Command: GenerateSkillGapAnalysis

```
Command: GenerateSkillGapAnalysis
  Input: { targetType, targetId, jobRoleId }
  
  Processing:
    1. Load current skill levels from rm_user_skills projection
    2. Load required skill levels from job_role_skills
    3. Calculate gaps for each skill
    4. Prioritize gaps by size and skill importance
    5. Generate recommended learning pathways (AI)
  
  Events produced:
    -> SkillGapAnalysisGenerated { analysisId, targetType, targetId, gaps[], overallReadiness }
    -> For each critical gap with available courses:
       -> LearningPathwayAssigned { pathwayId, userId, steps[] } (AI-generated pathway)
```

---

## Event Processing Pipeline

```
                    ┌──────────────┐
                    │   Commands   │
                    │  (Write API) │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  Event Store │
                    │ (PostgreSQL) │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  Event Bus   │
                    │   (Kafka)    │
                    └──┬───┬───┬───┘
                       │   │   │
            ┌──────────┘   │   └──────────┐
            │              │              │
     ┌──────▼─────┐ ┌─────▼──────┐ ┌─────▼──────┐
     │ Enrolment  │ │   Skills   │ │ Compliance │
     │ Projection │ │ Projection │ │ Projection │
     └──────┬─────┘ └─────┬──────┘ └─────┬──────┘
            │              │              │
     ┌──────▼─────┐ ┌─────▼──────┐ ┌─────▼──────┐
     │rm_enrolments│ │rm_user_    │ │rm_compliance│
     │            │ │  skills    │ │  _status   │
     └────────────┘ └────────────┘ └────────────┘
                           │
                    ┌──────▼───────┐
                    │  Read API    │
                    │  (Queries)   │
                    └──────────────┘
```

---

## Pros and Cons

### Pros

1. **Complete audit trail by design**: Every state change is an immutable event. Compliance auditors can replay the exact sequence of actions for any learner, any course, any time period. There is no separate audit_log table to maintain — the event store IS the audit log.

2. **Natural fit for xAPI**: xAPI statements are inherently events (Actor-Verb-Object). An event-sourced platform stores xAPI natively as XApiStatementReceived events without needing a separate LRS data model. The event store becomes a standards-compliant Learning Record Store.

3. **Temporal queries for free**: "What was this learner's skill profile on March 1st?" is trivially answered by replaying events up to that date. In a traditional CRUD model, this requires a separate versioning system.

4. **Independent scaling**: The write side (event append) and read side (projections) scale independently. During peak enrolment periods, the write side handles the load while read projections catch up asynchronously.

5. **Projection flexibility**: New dashboard requirements or analytics needs can be met by adding new projections that replay the full event history. No schema migration needed for read-side changes.

6. **AI training data**: The complete event stream provides rich, sequential training data for AI models that predict skill gaps, recommend content, and detect learning pattern anomalies.

7. **Resilient integrations**: Integration with Slack, Teams, HRIS systems can be built as event consumers that react to specific events. If an integration fails, it can replay missed events.

### Cons

1. **Architectural complexity**: CQRS + Event Sourcing is significantly more complex than CRUD. The team needs expertise in event-driven architecture, eventual consistency, and distributed systems patterns.

2. **Eventual consistency**: Read models lag behind writes. A learner who just completed a course may not see their updated status immediately. This requires careful UX design with optimistic updates on the client side.

3. **Event schema evolution**: As the platform evolves, event schemas change. Old events must remain readable, requiring upcasters/versioning strategies for backward compatibility. Event type `CourseCompleted_v1` vs `CourseCompleted_v2` adds complexity.

4. **Projection rebuild time**: If a projection has a bug or a new projection is added, replaying millions of events can take hours or days. Snapshots help but add another layer of complexity.

5. **SCORM suspend_data challenge**: SCORM requires stateful, mutable runtime data (suspend_data updates on every LMSCommit). Event sourcing every commit generates enormous event volume. The ScormDataCommitted event may fire hundreds of times per learner per module.

6. **Debugging difficulty**: Tracing a bug requires following events through the store, through projections, and understanding the eventual consistency window. Standard SQL debugging is simpler.

7. **Storage growth**: Every state change produces an event. A 50,000-user organization with active learning could generate 1M+ events per day. While events compress well, storage costs are higher than a CRUD model that overwrites state.

8. **Team ramp-up**: Most developers are experienced with CRUD, not event sourcing. Hiring and onboarding are harder.

---

## Migration and Scaling Considerations

### Migration from Legacy Systems

1. **Event backfill**: Import historical SCORM completion records and xAPI statements as synthetic events with original timestamps. This populates projections with historical data from day one.

2. **Dual-write transition**: During migration, write to both old CRUD database and new event store. Validate projection outputs match old queries. Switch reads to projections once validated.

3. **Event versioning strategy**: Use an event upcasting pipeline that transforms old event formats to current format during replay. Store the original event immutably and apply transformations at read time.

### Scaling Strategy

1. **Event store partitioning**: Partition the event_store table by month. Archive partitions older than the retention window to cold storage (S3/Glacier). Keep recent months on fast SSDs.

2. **Kafka topics**: Separate topics per aggregate type (user-events, enrolment-events, skill-events) to allow independent consumer scaling.

3. **Projection parallelism**: Each projection runs as an independent consumer group. Multiple instances can process different partitions in parallel.

4. **Snapshots**: For aggregates with long event histories (e.g., a power user with 500+ enrolment events), store periodic snapshots to avoid replaying from the beginning.

5. **Read replica scaling**: Since projections are rebuilt tables, they can be replicated and load-balanced independently from the event store.

6. **Estimated throughput**: A well-tuned PostgreSQL event store on modern hardware can handle 10,000-50,000 event appends per second. Kafka can handle 100,000+ events per second per partition. For a 50,000-user L&D platform, peak write load during company-wide compliance training rollout might reach 5,000 events/second — well within capacity.

### Snapshot Strategy

```sql
-- Snapshots are stored in stream_metadata for fast aggregate loading
UPDATE stream_metadata
SET snapshot_version = 150,
    snapshot_data = '{
        "enrolment_id": "...",
        "status": "in_progress",
        "completion_pct": 65.0,
        "score": null,
        "time_spent_seconds": 7200,
        "attempt_count": 1,
        "modules_completed": ["mod1", "mod2", "mod3"],
        "current_module": "mod4"
    }'::jsonb,
    updated_at = now()
WHERE stream_id = 'enrolment-550e8400-e29b-41d4-a716-446655440000';

-- When loading an aggregate, start from snapshot and replay only newer events:
-- SELECT * FROM event_store
-- WHERE stream_id = 'enrolment-...' AND event_version > 150
-- ORDER BY event_version;
```
