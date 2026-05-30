# Data Model Suggestion 4: Graph Database + Polyglot Persistence

## Overview

This model uses a graph database (Neo4j) as the **primary intelligence layer** for skills ontology, competency relationships, learning pathway optimization, and organizational talent mapping — the domains where the corporate L&D platform derives its most distinctive value. The graph layer is complemented by purpose-selected storage engines for other domains: PostgreSQL for transactional LMS operations, an xAPI-compliant Learning Record Store for learning activity data, and Redis for real-time session state.

The rationale for a graph-centric approach is rooted in the observation that the highest-value operations in a corporate L&D platform are fundamentally graph problems:

- **Skills ontology traversal**: "What skills are prerequisites for this skill? What other skills are related? What role requires a cluster of these skills?" These are multi-hop graph traversals that are awkward and slow in relational databases but native to graph databases.
- **Learning pathway optimization**: "Given this learner's current skills and target role, what is the shortest/most effective sequence of courses to close the gap?" This is a shortest-path problem on a weighted graph.
- **Organizational talent mapping**: "Which teams have the deepest expertise in cloud architecture? Where are the single points of failure (only one person with a critical skill)?" These require graph analytics over the org-skills-people network.
- **AI-powered recommendation**: Graph embeddings from skill-course-learner relationships feed directly into recommendation models. Node2Vec or GraphSAGE on the skills graph produces vector representations useful for similarity search and gap prediction.
- **Impact propagation**: "If we upskill 10 people in Kubernetes, how does that change our overall cloud readiness score across all dependent skills?" This is influence propagation through a skill dependency graph.

---

## Technology Recommendations

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Graph Database | Neo4j 5.x (Community or Enterprise) | Most mature graph DB, native Cypher query language, APOC library, GDS (Graph Data Science) library for analytics |
| Transactional DB | PostgreSQL 16+ | ACID compliance for enrolments, compliance, and audit; SCORM runtime data |
| Learning Record Store | PostgreSQL with xAPI schema or standalone LRS (Learning Locker) | xAPI statement storage and querying |
| Cache / Session Store | Redis | SCORM suspend_data, active session state, real-time dashboards |
| Search Engine | OpenSearch | Full-text search across courses, skills, and content |
| Message Broker | Apache Kafka | Event streaming between components; synchronization between Neo4j and PostgreSQL |
| Vector Store | Neo4j native vector index or pgvector | Skill embeddings for AI-powered similarity and recommendations |
| Application Layer | Node.js/TypeScript or Python | Both have strong Neo4j drivers and ML/AI library ecosystems |

---

## Architecture Overview

```
┌───────────────────────────────────────────────────────────┐
│                    Application Layer                       │
│           (REST API / GraphQL / gRPC)                     │
└────┬──────────┬──────────┬──────────┬──────────┬──────────┘
     │          │          │          │          │
┌────▼────┐ ┌──▼───┐ ┌────▼────┐ ┌──▼───┐ ┌───▼────┐
│  Neo4j  │ │Postgre│ │  Redis  │ │OpenSea│ │ Kafka  │
│ (Graph) │ │ SQL   │ │ (Cache) │ │ rch   │ │(Events)│
│         │ │       │ │         │ │       │ │        │
│• Skills │ │•Enrol-│ │•SCORM   │ │•Course│ │•Sync   │
│  Ontolo-│ │ ments │ │ suspend │ │ search│ │ events │
│  gy     │ │•Compli│ │ data    │ │•Skill │ │•Change │
│• Pathway│ │ ance  │ │•Session │ │ search│ │ data   │
│  Graph  │ │•Audit │ │ state   │ │•xAPI  │ │ capture│
│• Org    │ │•xAPI  │ │•Dash-   │ │ query │ │        │
│  Talent │ │ Store │ │ board   │ │       │ │        │
│  Map    │ │•Users │ │ cache   │ │       │ │        │
│• AI     │ │•KPIs  │ │         │ │       │ │        │
│  Recomm │ │       │ │         │ │       │ │        │
└─────────┘ └───────┘ └─────────┘ └───────┘ └────────┘
```

---

## Neo4j Graph Schema

### Node Types and Properties

```cypher
// =============================================================
// TENANT
// =============================================================
CREATE CONSTRAINT tenant_id_unique IF NOT EXISTS
FOR (t:Tenant) REQUIRE t.tenantId IS UNIQUE;

// Properties:
// - tenantId: STRING (UUID)
// - name: STRING
// - slug: STRING
// - isActive: BOOLEAN
// - createdAt: DATETIME

// =============================================================
// ORGANIZATION (department, division, team)
// =============================================================
CREATE CONSTRAINT org_id_unique IF NOT EXISTS
FOR (o:Organization) REQUIRE o.orgId IS UNIQUE;

// Properties:
// - orgId: STRING (UUID)
// - tenantId: STRING
// - name: STRING
// - orgType: STRING ('company', 'division', 'department', 'team')
// - code: STRING
// - isActive: BOOLEAN

// =============================================================
// PERSON (user/employee)
// =============================================================
CREATE CONSTRAINT person_id_unique IF NOT EXISTS
FOR (p:Person) REQUIRE p.userId IS UNIQUE;

CREATE INDEX person_email IF NOT EXISTS
FOR (p:Person) ON (p.email);

CREATE INDEX person_tenant IF NOT EXISTS
FOR (p:Person) ON (p.tenantId);

// Properties:
// - userId: STRING (UUID)
// - tenantId: STRING
// - username: STRING
// - email: STRING
// - firstName: STRING
// - lastName: STRING
// - displayName: STRING
// - jobTitle: STRING
// - status: STRING ('active', 'inactive')
// - lastLoginAt: DATETIME

// =============================================================
// SKILL
// =============================================================
CREATE CONSTRAINT skill_id_unique IF NOT EXISTS
FOR (s:Skill) REQUIRE s.skillId IS UNIQUE;

CREATE INDEX skill_name IF NOT EXISTS
FOR (s:Skill) ON (s.name);

CREATE FULLTEXT INDEX skill_search IF NOT EXISTS
FOR (s:Skill) ON EACH [s.name, s.description];

// Properties:
// - skillId: STRING (UUID)
// - frameworkId: STRING
// - name: STRING
// - description: STRING
// - skillType: STRING ('technical', 'soft', 'transversal', 'domain', 'certification')
// - escoUri: STRING (optional — ESCO concept URI)
// - onetElementId: STRING (optional — O*NET element ID)
// - isActive: BOOLEAN
// Additional labels: :TechnicalSkill, :SoftSkill, :TransversalSkill (multi-label for filtering)

// =============================================================
// SKILL CATEGORY
// =============================================================
CREATE CONSTRAINT category_id_unique IF NOT EXISTS
FOR (c:SkillCategory) REQUIRE c.categoryId IS UNIQUE;

// Properties:
// - categoryId: STRING (UUID)
// - frameworkId: STRING
// - name: STRING
// - code: STRING
// - level: INTEGER (depth in hierarchy)

// =============================================================
// SKILL FRAMEWORK
// =============================================================
CREATE CONSTRAINT framework_id_unique IF NOT EXISTS
FOR (f:SkillFramework) REQUIRE f.frameworkId IS UNIQUE;

// Properties:
// - frameworkId: STRING (UUID)
// - tenantId: STRING
// - name: STRING
// - source: STRING ('esco', 'onet', 'custom')
// - version: STRING

// =============================================================
// JOB ROLE
// =============================================================
CREATE CONSTRAINT jobrole_id_unique IF NOT EXISTS
FOR (r:JobRole) REQUIRE r.roleId IS UNIQUE;

// Properties:
// - roleId: STRING (UUID)
// - tenantId: STRING
// - name: STRING
// - level: STRING ('junior', 'mid', 'senior', 'lead', 'principal')
// - jobFamily: STRING
// - onetCode: STRING

// =============================================================
// COURSE
// =============================================================
CREATE CONSTRAINT course_id_unique IF NOT EXISTS
FOR (c:Course) REQUIRE c.courseId IS UNIQUE;

CREATE FULLTEXT INDEX course_search IF NOT EXISTS
FOR (c:Course) ON EACH [c.title, c.description];

// Properties:
// - courseId: STRING (UUID)
// - tenantId: STRING
// - title: STRING
// - description: STRING
// - contentType: STRING
// - difficultyLevel: STRING
// - estimatedDurationMinutes: INTEGER
// - qualityScore: FLOAT
// - status: STRING
// - isCompliance: BOOLEAN

// =============================================================
// LEARNING PATHWAY
// =============================================================
CREATE CONSTRAINT pathway_id_unique IF NOT EXISTS
FOR (lp:LearningPathway) REQUIRE lp.pathwayId IS UNIQUE;

// Properties:
// - pathwayId: STRING (UUID)
// - tenantId: STRING
// - title: STRING
// - pathwayType: STRING
// - estimatedDurationDays: INTEGER
// - status: STRING
// - optimizationScore: FLOAT (AI-computed pathway quality)

// =============================================================
// CERTIFICATION
// =============================================================
CREATE CONSTRAINT cert_id_unique IF NOT EXISTS
FOR (cert:Certification) REQUIRE cert.certificationId IS UNIQUE;

// Properties:
// - certificationId: STRING (UUID)
// - tenantId: STRING
// - name: STRING
// - issuingBody: STRING
// - validityDays: INTEGER

// =============================================================
// PROFICIENCY LEVEL (reusable node for skill level definitions)
// =============================================================
// Properties:
// - level: INTEGER (1-10)
// - label: STRING ('Awareness', 'Novice', ...)
// - description: STRING
```

### Relationship Types

```cypher
// =============================================================
// ORGANIZATIONAL HIERARCHY
// =============================================================

// Organization tree
// (:Organization)-[:PART_OF]->(:Organization)
// Properties: since (DATE)

// Person belongs to organization
// (:Person)-[:BELONGS_TO]->(:Organization)
// Properties: since (DATE), role (STRING)

// Manager relationship
// (:Person)-[:REPORTS_TO]->(:Person)
// Properties: since (DATE)

// Tenant ownership
// (:Organization)-[:OWNED_BY]->(:Tenant)
// (:Person)-[:MEMBER_OF]->(:Tenant)

// =============================================================
// SKILLS ONTOLOGY RELATIONSHIPS
// =============================================================

// Skill hierarchy
// (:Skill)-[:SUBCATEGORY_OF]->(:SkillCategory)
// (:SkillCategory)-[:CHILD_OF]->(:SkillCategory)
// (:SkillCategory)-[:PART_OF]->(:SkillFramework)

// Skill-to-skill relationships (THE CORE VALUE OF THE GRAPH MODEL)
// (:Skill)-[:PREREQUISITE_FOR {strength: 0.9}]->(:Skill)
// (:Skill)-[:RELATED_TO {strength: 0.7, source: 'ai_inferred', confidence: 0.85}]->(:Skill)
// (:Skill)-[:SUPERSEDES]->(:Skill)
// (:Skill)-[:COMPONENT_OF]->(:Skill)    // e.g., "React" is component of "Frontend Development"
// (:Skill)-[:ALTERNATIVE_TO]->(:Skill)  // e.g., "Angular" is alternative to "React"

// =============================================================
// PERSON-SKILL RELATIONSHIPS (competency graph)
// =============================================================

// Current skill level
// (:Person)-[:HAS_SKILL {
//     currentLevel: 5,
//     targetLevel: 7,
//     assessmentMethod: 'certification',
//     assessedAt: datetime(),
//     confidence: 0.95,
//     evidenceUrl: 'https://...'
// }]->(:Skill)

// Skill development history (temporal relationships)
// (:Person)-[:SKILL_ASSESSED {
//     level: 3,
//     method: 'self',
//     assessedAt: datetime('2025-06-01'),
//     assessedBy: 'user-uuid'
// }]->(:Skill)
// Multiple SKILL_ASSESSED relationships track history

// =============================================================
// JOB ROLE - SKILL REQUIREMENTS
// =============================================================

// (:JobRole)-[:REQUIRES_SKILL {
//     requiredLevel: 7,
//     importance: 'critical'  // critical, required, preferred, nice_to_have
// }]->(:Skill)

// Career progression
// (:JobRole)-[:PROGRESSES_TO {typical_years: 2}]->(:JobRole)
// (:JobRole)-[:PREREQUISITE_ROLE_FOR]->(:JobRole)

// Person's current role
// (:Person)-[:HOLDS_ROLE {since: date()}]->(:JobRole)

// =============================================================
// COURSE-SKILL RELATIONSHIPS
// =============================================================

// (:Course)-[:TEACHES_SKILL {
//     levelTaught: 5,
//     isPrimary: true,
//     confidence: 0.9        // how well does this course actually teach this skill
// }]->(:Skill)

// Course prerequisites
// (:Course)-[:PREREQUISITE_FOR]->(:Course)

// =============================================================
// LEARNING PATHWAY GRAPH
// =============================================================

// (:LearningPathway)-[:TARGETS_ROLE]->(:JobRole)
// (:LearningPathway)-[:ADDRESSES_GAP]->(:Skill)

// Pathway steps as relationships (ordered)
// (:LearningPathway)-[:STARTS_WITH]->(:Course)
// (:Course)-[:FOLLOWED_BY {stepOrder: 2, isRequired: true}]->(:Course)
// (:Course)-[:FOLLOWED_BY {stepOrder: 3, isRequired: false}]->(:Course)

// Alternative: Step nodes for more complex pathway logic
// (:LearningPathway)-[:HAS_STEP]->(:PathwayStep {order: 1, type: 'course'})
// (:PathwayStep)-[:REFERENCES]->(:Course)
// (:PathwayStep)-[:NEXT_STEP]->(:PathwayStep)
// (:PathwayStep)-[:REQUIRES_COMPLETION_OF]->(:PathwayStep)

// =============================================================
// CERTIFICATION RELATIONSHIPS
// =============================================================

// (:Certification)-[:VALIDATES_SKILL {level: 7}]->(:Skill)
// (:Course)-[:LEADS_TO_CERTIFICATION]->(:Certification)
// (:Person)-[:HOLDS_CERTIFICATION {
//     issuedAt: datetime(),
//     expiresAt: datetime(),
//     certificateNumber: 'CERT-2026-001',
//     status: 'active'
// }]->(:Certification)
```

### Key Cypher Queries

```cypher
// =============================================================
// QUERY 1: Skill Gap Analysis for a Person targeting a Job Role
// (The query that makes graph databases shine)
// =============================================================
MATCH (p:Person {userId: $userId})-[:HOLDS_ROLE]->(currentRole:JobRole)
MATCH (targetRole:JobRole {roleId: $targetRoleId})
MATCH (targetRole)-[req:REQUIRES_SKILL]->(skill:Skill)
OPTIONAL MATCH (p)-[has:HAS_SKILL]->(skill)
WITH skill, req, has,
     req.requiredLevel AS required,
     COALESCE(has.currentLevel, 0) AS current,
     req.importance AS importance
WHERE current < required
RETURN
    skill.name AS skillName,
    skill.skillType AS skillType,
    required AS requiredLevel,
    current AS currentLevel,
    required - current AS gap,
    importance,
    skill.skillId AS skillId
ORDER BY
    CASE importance
        WHEN 'critical' THEN 1
        WHEN 'required' THEN 2
        WHEN 'preferred' THEN 3
        ELSE 4
    END,
    (required - current) DESC;

// =============================================================
// QUERY 2: AI-Powered Learning Pathway Generation
// Find the optimal sequence of courses to close skill gaps
// (Weighted shortest path through the course-skill graph)
// =============================================================
// Step 1: Identify gaps
MATCH (p:Person {userId: $userId})
MATCH (targetRole:JobRole {roleId: $targetRoleId})-[req:REQUIRES_SKILL]->(skill:Skill)
OPTIONAL MATCH (p)-[has:HAS_SKILL]->(skill)
WITH p, skill, req.requiredLevel AS required, COALESCE(has.currentLevel, 0) AS current
WHERE current < required
WITH collect({skill: skill, gap: required - current, required: required}) AS gaps

// Step 2: Find courses that address these gaps
UNWIND gaps AS g
MATCH (course:Course {status: 'published'})-[teaches:TEACHES_SKILL]->(g.skill)
WHERE teaches.levelTaught >= g.required
WITH g.skill AS skill, g.gap AS gap, course,
     course.qualityScore AS quality,
     course.estimatedDurationMinutes AS duration
ORDER BY quality DESC, duration ASC
WITH skill, gap, collect(course)[0] AS bestCourse  // pick best course per skill

// Step 3: Order by prerequisite chain
OPTIONAL MATCH (bestCourse)-[:PREREQUISITE_FOR]->(nextCourse)
RETURN
    skill.name AS skillName,
    gap AS skillGap,
    bestCourse.title AS courseTitle,
    bestCourse.courseId AS courseId,
    bestCourse.estimatedDurationMinutes AS durationMinutes,
    bestCourse.qualityScore AS qualityScore
ORDER BY gap DESC;

// =============================================================
// QUERY 3: Team Skill Heatmap
// =============================================================
MATCH (org:Organization {orgId: $orgId})<-[:BELONGS_TO]-(p:Person {status: 'active'})
WITH org, collect(p) AS teamMembers, count(p) AS teamSize
UNWIND teamMembers AS member
OPTIONAL MATCH (member)-[has:HAS_SKILL]->(skill:Skill)
WITH org, skill, teamSize,
     count(has) AS assessedCount,
     avg(has.currentLevel) AS avgLevel,
     min(has.currentLevel) AS minLevel,
     max(has.currentLevel) AS maxLevel
WHERE skill IS NOT NULL
RETURN
    org.name AS teamName,
    skill.name AS skillName,
    skill.skillType AS skillType,
    avgLevel,
    minLevel,
    maxLevel,
    assessedCount,
    teamSize,
    toFloat(assessedCount) / teamSize * 100 AS coveragePct
ORDER BY avgLevel ASC;

// =============================================================
// QUERY 4: Find Skill Clusters (skills that frequently co-occur)
// This powers AI-driven skill inference
// =============================================================
MATCH (s1:Skill)<-[:HAS_SKILL]-(p:Person)-[:HAS_SKILL]->(s2:Skill)
WHERE s1.skillId < s2.skillId  // avoid duplicates
    AND p.tenantId = $tenantId
WITH s1, s2, count(p) AS coOccurrence
WHERE coOccurrence > 5  // minimum support threshold
RETURN
    s1.name AS skill1,
    s2.name AS skill2,
    coOccurrence,
    // Check if explicit relationship already exists
    EXISTS((s1)-[:RELATED_TO]-(s2)) AS alreadyLinked
ORDER BY coOccurrence DESC
LIMIT 50;

// =============================================================
// QUERY 5: Prerequisite Chain Validation
// Before enrolling a learner, check if they have all prerequisites
// =============================================================
MATCH path = (course:Course {courseId: $courseId})<-[:PREREQUISITE_FOR*1..5]-(prereq:Course)
WITH prereq, length(path) AS depth
MATCH (p:Person {userId: $userId})
OPTIONAL MATCH (p)-[e:ENROLLED_IN {status: 'completed'}]->(prereq)
RETURN
    prereq.title AS prerequisiteCourse,
    prereq.courseId AS prereqId,
    depth AS chainDepth,
    CASE WHEN e IS NOT NULL THEN true ELSE false END AS completed
ORDER BY depth DESC;

// =============================================================
// QUERY 6: Organizational Talent Risk Analysis
// Find critical skills where only 1-2 people have expertise
// =============================================================
MATCH (org:Organization {orgId: $orgId})<-[:BELONGS_TO*1..3]-(subOrg:Organization)
    <-[:BELONGS_TO]-(p:Person {status: 'active'})-[has:HAS_SKILL]->(skill:Skill)
WHERE has.currentLevel >= 7  // expert level
WITH skill, collect(DISTINCT p) AS experts, count(DISTINCT p) AS expertCount
WHERE expertCount <= 2
MATCH (role:JobRole)-[req:REQUIRES_SKILL {importance: 'critical'}]->(skill)
RETURN
    skill.name AS criticalSkill,
    expertCount AS numberOfExperts,
    [e IN experts | e.displayName] AS expertNames,
    collect(DISTINCT role.name) AS rolesRequiring
ORDER BY expertCount ASC;

// =============================================================
// QUERY 7: Career Path Exploration
// Show all possible career paths from current role
// =============================================================
MATCH (p:Person {userId: $userId})-[:HOLDS_ROLE]->(current:JobRole)
MATCH path = (current)-[:PROGRESSES_TO*1..4]->(future:JobRole)
WITH path, future, length(path) AS steps,
     [n IN nodes(path) | n.name] AS pathNames
// For each future role, calculate readiness
MATCH (future)-[req:REQUIRES_SKILL]->(skill:Skill)
OPTIONAL MATCH (p)-[has:HAS_SKILL]->(skill)
WITH future, steps, pathNames,
     count(skill) AS totalSkills,
     count(CASE WHEN COALESCE(has.currentLevel, 0) >= req.requiredLevel THEN 1 END) AS metSkills
RETURN
    future.name AS targetRole,
    future.level AS roleLevel,
    steps AS stepsAway,
    pathNames AS careerPath,
    totalSkills,
    metSkills,
    toFloat(metSkills) / totalSkills * 100 AS readinessPct
ORDER BY steps ASC, readinessPct DESC;

// =============================================================
// QUERY 8: Impact Propagation — Upskilling ROI Estimation
// If we train N people in skill X, how does it affect
// dependent skills and overall capability?
// =============================================================
MATCH (targetSkill:Skill {skillId: $skillId})
MATCH path = (targetSkill)<-[:COMPONENT_OF|PREREQUISITE_FOR*1..3]-(dependent:Skill)
WITH dependent, length(path) AS distance,
     1.0 / (1 + length(path)) AS impactFactor  // decay by distance
MATCH (org:Organization {orgId: $orgId})<-[:BELONGS_TO]-(p:Person)
OPTIONAL MATCH (p)-[has:HAS_SKILL]->(dependent)
WITH dependent, distance, impactFactor,
     count(p) AS teamSize,
     count(has) AS withSkill,
     avg(COALESCE(has.currentLevel, 0)) AS currentAvg
RETURN
    dependent.name AS impactedSkill,
    distance AS degreesAway,
    impactFactor,
    teamSize,
    withSkill,
    currentAvg,
    currentAvg * (1 + impactFactor * 0.2) AS projectedAvgAfterTraining
ORDER BY impactFactor DESC;
```

### Graph Data Science (GDS) Analytics

```cypher
// =============================================================
// PageRank on Skills — Which skills are most central?
// (Skills that are prerequisites for many other skills)
// =============================================================
CALL gds.pageRank.stream('skills-graph', {
    nodeLabels: ['Skill'],
    relationshipTypes: ['PREREQUISITE_FOR', 'COMPONENT_OF'],
    maxIterations: 50,
    dampingFactor: 0.85
})
YIELD nodeId, score
WITH gds.util.asNode(nodeId) AS skill, score
RETURN skill.name AS skillName, skill.skillType, score AS centrality
ORDER BY score DESC
LIMIT 20;

// =============================================================
// Community Detection — Find natural skill clusters
// =============================================================
CALL gds.louvain.stream('skills-graph', {
    nodeLabels: ['Skill'],
    relationshipTypes: ['RELATED_TO', 'COMPONENT_OF']
})
YIELD nodeId, communityId
WITH gds.util.asNode(nodeId) AS skill, communityId
RETURN communityId, collect(skill.name) AS skillsInCluster, count(*) AS clusterSize
ORDER BY clusterSize DESC;

// =============================================================
// Node Similarity — Find skills similar to a given skill
// (powers "if you know X, you might also know Y" recommendations)
// =============================================================
CALL gds.nodeSimilarity.stream('people-skills-graph', {
    nodeLabels: ['Skill'],
    relationshipTypes: ['HAS_SKILL'],
    topK: 10
})
YIELD node1, node2, similarity
WITH gds.util.asNode(node1) AS skill1, gds.util.asNode(node2) AS skill2, similarity
WHERE skill1.skillId = $skillId
RETURN skill2.name AS similarSkill, similarity
ORDER BY similarity DESC;

// =============================================================
// Vector Embeddings for AI Recommendations
// Neo4j 5.x supports native vector indexes
// =============================================================
// Store skill embeddings (generated by GraphSAGE or Node2Vec)
MATCH (s:Skill {skillId: $skillId})
SET s.embedding = $embeddingVector;

// Create vector index
CREATE VECTOR INDEX skill_embeddings IF NOT EXISTS
FOR (s:Skill) ON (s.embedding)
OPTIONS {indexConfig: {
    `vector.dimensions`: 128,
    `vector.similarity_function`: 'cosine'
}};

// Query similar skills by vector similarity
CALL db.index.vector.queryNodes('skill_embeddings', 10, $queryVector)
YIELD node, score
RETURN node.name AS skillName, node.skillType, score AS similarity;
```

---

## PostgreSQL Schema (Transactional Layer)

The PostgreSQL database handles transactional operations where ACID guarantees and relational integrity are essential.

```sql
-- =============================================================
-- USERS (authoritative source, synced to Neo4j as Person nodes)
-- =============================================================
CREATE TABLE users (
    user_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    username        VARCHAR(255) NOT NULL,
    email           VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    display_name    VARCHAR(255),
    job_title       VARCHAR(255),
    org_id          UUID,
    manager_id      UUID REFERENCES users(user_id),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    profile_json    JSONB NOT NULL DEFAULT '{}',
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Sync tracking
    neo4j_synced_at TIMESTAMPTZ,
    UNIQUE (tenant_id, username),
    UNIQUE (tenant_id, email)
);

-- =============================================================
-- ENROLMENTS
-- =============================================================
CREATE TABLE enrolments (
    enrolment_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID NOT NULL REFERENCES users(user_id),
    course_id       UUID NOT NULL,                  -- references Neo4j Course node
    pathway_id      UUID,
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
    assigned_by     UUID REFERENCES users(user_id),
    compliance_rule_id UUID,
    context_json    JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_enrolments_user ON enrolments (user_id);
CREATE INDEX idx_enrolments_course ON enrolments (course_id);
CREATE INDEX idx_enrolments_status ON enrolments (tenant_id, status);
CREATE INDEX idx_enrolments_compliance ON enrolments (due_date)
    WHERE status NOT IN ('completed', 'withdrawn');

-- =============================================================
-- SCORM RUNTIME DATA
-- =============================================================
CREATE TABLE scorm_runtime (
    runtime_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    enrolment_id    UUID NOT NULL REFERENCES enrolments(enrolment_id) ON DELETE CASCADE,
    sco_identifier  VARCHAR(255) NOT NULL,
    attempt_number  INTEGER NOT NULL DEFAULT 1,
    lesson_status   VARCHAR(30),
    score_scaled    NUMERIC(5,4),
    score_raw       NUMERIC(10,4),
    total_time      VARCHAR(50),
    cmi_data        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (enrolment_id, sco_identifier, attempt_number)
);

-- =============================================================
-- xAPI STATEMENTS (LRS)
-- =============================================================
CREATE TABLE xapi_statements (
    statement_id    UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    actor_user_id   UUID REFERENCES users(user_id),
    verb_id         VARCHAR(500) NOT NULL,
    object_id       VARCHAR(500) NOT NULL,
    result_score_scaled NUMERIC(5,4),
    result_success  BOOLEAN,
    result_completion BOOLEAN,
    context_registration UUID,
    timestamp       TIMESTAMPTZ NOT NULL,
    stored          TIMESTAMPTZ NOT NULL DEFAULT now(),
    voided          BOOLEAN NOT NULL DEFAULT FALSE,
    statement_json  JSONB NOT NULL
) PARTITION BY RANGE (stored);

CREATE INDEX idx_xapi_tenant ON xapi_statements (tenant_id, stored DESC);
CREATE INDEX idx_xapi_actor ON xapi_statements (actor_user_id);
CREATE INDEX idx_xapi_verb ON xapi_statements (verb_id);

-- =============================================================
-- COMPLIANCE RULES
-- =============================================================
CREATE TABLE compliance_rules (
    rule_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    course_id       UUID NOT NULL,
    rule_config     JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =============================================================
-- BUSINESS KPIs
-- =============================================================
CREATE TABLE business_kpi_values (
    value_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    kpi_name        VARCHAR(255) NOT NULL,
    kpi_category    VARCHAR(50) NOT NULL,
    user_id         UUID REFERENCES users(user_id),
    org_id          UUID,
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    value           NUMERIC(15,4) NOT NULL,
    context_json    JSONB NOT NULL DEFAULT '{}',
    imported_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_kpi_values_period ON business_kpi_values (kpi_name, period_start);

-- =============================================================
-- NOTIFICATIONS
-- =============================================================
CREATE TABLE notifications (
    notification_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID NOT NULL REFERENCES users(user_id),
    channel         VARCHAR(20) NOT NULL,
    notification_type VARCHAR(50) NOT NULL,
    content_json    JSONB NOT NULL DEFAULT '{}',
    is_read         BOOLEAN NOT NULL DEFAULT FALSE,
    sent_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    read_at         TIMESTAMPTZ
);

CREATE INDEX idx_notifications_user ON notifications (user_id, sent_at DESC);

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
    changes_json    JSONB NOT NULL DEFAULT '{}',
    request_json    JSONB NOT NULL DEFAULT '{}',
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

CREATE INDEX idx_audit_tenant ON audit_log (tenant_id, occurred_at DESC);
CREATE INDEX idx_audit_resource ON audit_log (resource_type, resource_id);
```

---

## Data Synchronization Between Neo4j and PostgreSQL

```
┌──────────────┐    CDC / Kafka     ┌──────────────┐
│  PostgreSQL  │ ──────────────────>│    Neo4j     │
│              │                    │              │
│  users       │  UserCreated       │  :Person     │
│  enrolments  │  EnrolmentCompleted│  :ENROLLED_IN│
│              │  SkillAssessed     │  :HAS_SKILL  │
└──────────────┘                    └──────────────┘
                                           │
                                    Graph Analytics
                                           │
                                    ┌──────▼──────┐
                                    │  AI/ML      │
                                    │  Models     │
                                    │             │
                                    │ • Pathway   │
                                    │   optimiz.  │
                                    │ • Skill gap │
                                    │   predict.  │
                                    │ • Content   │
                                    │   recomm.   │
                                    └─────────────┘
```

### Synchronization Events (via Kafka)

```json
// Topic: ld-platform.users
{
    "event": "user.created",
    "timestamp": "2026-05-25T10:00:00Z",
    "payload": {
        "userId": "...",
        "tenantId": "...",
        "displayName": "Jane Smith",
        "orgId": "...",
        "managerId": "...",
        "jobTitle": "Senior Engineer"
    }
}

// Topic: ld-platform.enrolments
{
    "event": "enrolment.completed",
    "timestamp": "2026-05-25T16:00:00Z",
    "payload": {
        "enrolmentId": "...",
        "userId": "...",
        "courseId": "...",
        "score": 92.3,
        "completedAt": "2026-05-25T16:00:00Z"
    }
}

// Topic: ld-platform.skills
{
    "event": "skill.assessed",
    "timestamp": "2026-05-25T10:30:00Z",
    "payload": {
        "userId": "...",
        "skillId": "...",
        "previousLevel": 3,
        "newLevel": 5,
        "method": "certification"
    }
}
```

### Sync Consumer (Pseudocode)

```
// Neo4j Sync Consumer
on UserCreated(event):
    MERGE (p:Person {userId: event.userId})
    SET p.displayName = event.displayName,
        p.tenantId = event.tenantId,
        p.jobTitle = event.jobTitle
    WITH p
    MATCH (org:Organization {orgId: event.orgId})
    MERGE (p)-[:BELONGS_TO]->(org)
    WITH p
    OPTIONAL MATCH (mgr:Person {userId: event.managerId})
    WHERE event.managerId IS NOT NULL
    MERGE (p)-[:REPORTS_TO]->(mgr)

on EnrolmentCompleted(event):
    MATCH (p:Person {userId: event.userId})
    MATCH (c:Course {courseId: event.courseId})
    MERGE (p)-[e:COMPLETED]->(c)
    SET e.score = event.score,
        e.completedAt = datetime(event.completedAt)
    // Trigger skill level re-evaluation
    WITH p, c
    MATCH (c)-[teaches:TEACHES_SKILL]->(skill:Skill)
    MERGE (p)-[has:HAS_SKILL]->(skill)
    SET has.currentLevel = CASE
            WHEN has.currentLevel IS NULL THEN teaches.levelTaught
            WHEN teaches.levelTaught > has.currentLevel THEN teaches.levelTaught
            ELSE has.currentLevel
        END,
        has.assessedAt = datetime(),
        has.assessmentMethod = 'course_completion'

on SkillAssessed(event):
    MATCH (p:Person {userId: event.userId})
    MATCH (s:Skill {skillId: event.skillId})
    MERGE (p)-[has:HAS_SKILL]->(s)
    SET has.currentLevel = event.newLevel,
        has.assessmentMethod = event.method,
        has.assessedAt = datetime(event.timestamp)
    // Also create a historical assessment relationship
    CREATE (p)-[:SKILL_ASSESSED {
        level: event.newLevel,
        method: event.method,
        assessedAt: datetime(event.timestamp)
    }]->(s)
```

---

## Pros and Cons

### Pros

1. **Superior skills intelligence**: Graph traversal queries that would require complex recursive CTEs in SQL (prerequisite chains, skill clustering, career path exploration, impact propagation) execute in milliseconds in Neo4j. This is the core value proposition of the platform.

2. **Natural AI/ML integration**: Graph embeddings (Node2Vec, GraphSAGE) produce skill vectors that feed directly into recommendation models. Neo4j's native vector index enables similarity search without a separate vector database.

3. **Organizational talent mapping**: Multi-hop queries across people-skills-roles-organizations are what graph databases were built for. "Find all people within 3 management levels who have expert-level Kubernetes skills" is one Cypher query.

4. **Learning pathway optimization**: Finding the optimal course sequence to close skill gaps is a weighted shortest-path problem — a fundamental graph algorithm. Neo4j's GDS library provides PageRank, Louvain community detection, and Dijkstra's algorithm out of the box.

5. **ESCO/O*NET natural fit**: Skills taxonomies with hierarchical categories, cross-references, and multi-language labels are inherently graph-structured. Importing ESCO's RDF/SKOS data into Neo4j preserves its native semantics.

6. **Each store is used for its strength**: PostgreSQL handles ACID transactions, SCORM runtime state, and audit logs. Neo4j handles relationship-heavy queries. Redis handles real-time session state. Each component operates in its sweet spot.

7. **Visualization**: Neo4j Bloom and similar tools provide out-of-the-box graph visualization for skill ontologies, career paths, and organizational maps — valuable for L&D team dashboards and executive presentations.

### Cons

1. **Operational complexity**: Running and maintaining Neo4j, PostgreSQL, Redis, Kafka, and OpenSearch is significantly more complex than a single-database approach. Each requires monitoring, backup, scaling, and expertise.

2. **Data synchronization challenges**: Keeping Neo4j and PostgreSQL in sync requires a reliable event pipeline (Kafka). Network failures, consumer lag, and ordering issues can cause temporary inconsistency.

3. **Transaction boundaries**: Cross-database transactions are not possible. An enrolment that writes to PostgreSQL and Neo4j cannot be atomic. Saga patterns or eventual consistency must be accepted.

4. **Neo4j licensing costs**: Neo4j Enterprise (required for clustering, advanced security, and some GDS features) has significant licensing costs. The Community edition is more limited. AuraDB (managed cloud) is priced per GB of data.

5. **Team expertise**: Few developers are proficient in Cypher and graph data modeling. Hiring and training costs are higher than for a PostgreSQL-only approach.

6. **Write performance**: Neo4j's write throughput is lower than PostgreSQL's. Bulk imports (loading 13,000 ESCO skills) require the Neo4j import tool rather than standard Cypher, and relationship-heavy writes are slower than relational inserts.

7. **Testing complexity**: Integration tests must verify behavior across multiple databases. Test fixtures must set up consistent state in both Neo4j and PostgreSQL.

8. **Vendor lock-in risk**: Neo4j's Cypher query language is proprietary (though openCypher is partially standardized). Migrating away from Neo4j to another graph database (e.g., Amazon Neptune, TigerGraph) requires significant query rewriting.

---

## Migration and Scaling Considerations

### Migration Strategy

1. **Graph-first import**: Import ESCO and O*NET data directly into Neo4j from their RDF/CSV distributions. ESCO provides SKOS/RDF files that map naturally to graph nodes and relationships. O*NET provides CSV files for occupations, skills, and abilities.

2. **PostgreSQL-first users**: User data originates in PostgreSQL (SCIM sync, SSO) and is published to Neo4j via Kafka. Never write user data directly to Neo4j.

3. **Gradual enrichment**: Start with skills and courses in Neo4j. Add organizational mapping after core LMS is stable. Add AI/ML features (embeddings, recommendations) after sufficient learning data accumulates.

4. **Legacy LMS migration**: For organizations migrating from Moodle or SCORM-only systems, import historical completion records into PostgreSQL and derive skill assessments for Neo4j.

### Scaling Strategy

1. **Neo4j scaling**: Neo4j Enterprise supports causal clustering (read replicas). For a 50,000-user organization with ~20,000 skills and ~2,000 courses, the graph will have ~75,000 nodes and ~500,000+ relationships — comfortably within a single Neo4j instance (Neo4j handles billions of nodes). Read replicas handle concurrent dashboard queries.

2. **PostgreSQL scaling**: Standard read replicas for reporting. Partition xapi_statements and audit_log by month.

3. **Kafka scaling**: Add partitions to high-volume topics. Scale consumer groups independently.

4. **Redis scaling**: Redis Cluster for session data if needed, though a single Redis instance handles 100,000+ operations/second.

5. **Estimated resource requirements**:
   - Neo4j: 16GB RAM, 4 cores, 100GB SSD for 50K-user deployment
   - PostgreSQL: 32GB RAM, 8 cores, 500GB SSD (xAPI is the largest table)
   - Redis: 4GB RAM
   - Kafka: 3-node cluster, 100GB each
   - OpenSearch: 2-node cluster, 16GB RAM each

### Disaster Recovery

1. **Neo4j**: Online backup + transaction log shipping for point-in-time recovery. Neo4j Enterprise supports incremental backups.
2. **PostgreSQL**: WAL archiving + pg_basebackup for PITR.
3. **Kafka**: Topic replication factor of 3 across brokers.
4. **Redis**: AOF persistence + sentinel for failover. Redis data can be rebuilt from PostgreSQL if lost.
5. **Recovery order**: PostgreSQL first (authoritative), then Kafka replay to sync Neo4j. Skills ontology in Neo4j can be rebuilt from framework imports if needed.
