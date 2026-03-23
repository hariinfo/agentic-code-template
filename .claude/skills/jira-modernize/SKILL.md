---
name: gen-jira
description: |
  Generate JIRA-ready modernization stories from analysis, functional spec, and technical spec.
  Create an Epic → Story hierarchy with deep technical context, story estimates, acceptance criteria
  tied to both funcspec and techspec, migration/cutover stories, inter-story dependencies, and
  techspec section references (API-001, MSG-001, TD-001). Produces jira-stories.md with traceability
  header, git commit reference, fingerprint validation reminder, legacy file SHA-256 hashes, sprint
  breakdown table, and reverse-index mapping of legacy files to affected stories.
---

# JIRA Modernization Story Generator (gen-jira)

## Purpose

Transform functional and technical specifications into implementable JIRA stories for a modernization initiative. This skill bridges the gap between "what the legacy system does" (funcspec.md) and "how to build the replacement" (techspec.md) by producing stories that developers can immediately execute.

The output is a comprehensive `jira-stories.md` document that includes:

- **Epic → Story hierarchy** — logical grouping of work
- **Deep technical context** — direct references to techspec sections (API-001, MSG-001, TD-001) so developers don't have to hunt for implementation details
- **Dual acceptance criteria** — tied to functional requirements AND technical specifications
- **Complexity estimation** — S/M/L/XL informed by techspec detail
- **Migration/cutover stories** — not just greenfield development, but data migration and cutover orchestration
- **Dependencies** — explicit ordering constraints between stories
- **Traceability** — every story links to legacy source files with SHA-256 hashes for drift detection

## Trigger

```
/gen-jira <slug>
```

Where `<slug>` is the directory name under `modernization-output/` containing the prerequisite documents.

## Prerequisites

The skill requires three input documents in `modernization-output/{slug}/`:

1. **analysis.md** — Component inventory, inbound pathways, behavioral inventory, data flow diagram, and technology stack from `/analyze-entry`
2. **funcspec.md** — Technology-agnostic functional requirements with business actors, acceptance criteria (Given/When/Then), data model, and integration points from `/gen-funcspec`
3. **techspec.md** — Implementation-level technical specification with section references (API-001, MSG-001, DB-001, TD-001), transaction boundaries, SQL logic, message schemas, and migration constraints from `/gen-techspec`
4. **fingerprint.json** (optional but recommended) — Git commit hash and per-file SHA-256 hashes for drift detection and legacy file references

## Process

### Step 1: Read and Index All Three Specs

Load `analysis.md`, `funcspec.md`, and `techspec.md`. Build indices of:

- **Functional requirements** — extract every FR-NNN from funcspec.md with its description, acceptance criteria, and business context
- **Technical sections** — extract every section ID from techspec.md (e.g., API-001, MSG-001, DB-001, TD-001) with its implementation details and constraints
- **Legacy components** — from analysis.md, list every discovered file with its role (Presentation, Business Service, DAO, PLSQL, etc.)
- **Integration points** — from funcspec.md section 5, list all external systems, Kafka topics, and shared database tables
- **Behavioral rules** — from analysis.md behavioral inventory, extract every discrete behavior, validation, side effect, and error handling rule

### Step 2: Map Functional Requirements to Technical Sections

For each FR-NNN in funcspec.md:

1. Identify which techspec sections (API-NNN, MSG-NNN, DB-NNN, TD-NNN) implement it
2. Flag any FR without corresponding techspec detail — this indicates a gap requiring developer investigation
3. Build a lookup table: FR-NNN → [API-001, DB-003, TD-005, ...]

### Step 3: Identify Story Boundaries

Group related work into implementable stories. A single story should:

- Address one or more closely related FRs
- Fit within a sprint (roughly 5-13 story points)
- Have clear acceptance criteria that can be verified
- Have a specific legacy component or technical section as its primary focus

Common patterns:

- **Data Layer Story** — "Replace stored procedure X with JPA repository" (maps to DB-NNN and DAO-level FRs)
- **API Story** — "Implement REST endpoint for business operation Y" (maps to API-NNN and controller-level FRs)
- **Messaging Story** — "Produce/consume Kafka topic Z with new message schema" (maps to MSG-NNN and Kafka listener FRs)
- **Validation Story** — "Implement domain validation rules in service layer" (maps to validation FRs and DAO-level constraints)
- **Integration Story** — "Implement circuit breaker and fallback for external system call" (maps to integration point FRs)
- **Migration Story** — "Data migration for legacy table X: transform and validate" (maps to DB-NNN migration notes)
- **Cutover Story** — "Parallel run validation and cutover orchestration" (final validation and switchover)

### Step 4: Assign Complexity Estimates

For each story, estimate relative complexity (S/M/L/XL) based on:

- **Techspec detail level** — sections with 300+ lines of SQL, complex transaction logic, or multiple integration points → L or XL
- **Behavioral rules complexity** — the number of edge cases and error paths in behavioral inventory → increases estimate
- **Integration dependencies** — stories that interact with multiple external systems or require careful ordering → increase estimate
- **Migration burden** — data transformation and validation logic → L or XL
- **Testing complexity** — stories with many acceptance criteria or multiple success paths → increase estimate

Mapping:
- **S (Small)** — 2-3 days, low complexity, one integration point, <10 acceptance criteria
- **M (Medium)** — 5-8 days, moderate complexity, 1-2 integration points, 10-20 acceptance criteria
- **L (Large)** — 10-13 days, high complexity, 2-3 integration points, 20+ acceptance criteria, significant technical debt
- **XL (Extra Large)** — 2+ weeks, very high complexity, 3+ integration points, complex data migration, or dependent on other stories

### Step 5: Define Story Acceptance Criteria

For each story, create acceptance criteria that reference both funcspec and techspec:

**Pattern:**

```
- [ ] {Given precondition from funcspec}, When {user action or system event}, Then {outcome}
- [ ] {Technical acceptance from techspec} — verify {API contract | SQL query | message schema | transaction boundary}
- [ ] {Edge case from behavioral inventory or error handling}
- [ ] {Data validation or integrity constraint}
- [ ] {Performance or compliance requirement}
```

**Example:**

```
- [ ] Given an order with total > $10,000, When approval is requested,
      Then system applies MANAGER_APPROVAL rule before confirming (FR-003)
- [ ] Query performance: ORDER summary query (DB-002) returns within <100ms for
      1M record table (see techspec DB-002: Query Analysis)
- [ ] Concurrent approvals: two simultaneous approval requests for same order are
      serialized using SERIALIZABLE isolation (techspec TD-005: Transaction Boundaries)
- [ ] Backward compatibility: new API accepts legacy `order_number` field and maps to `order_id`
```

### Step 6: Identify Story Dependencies

Build a dependency graph:

- Story A depends on Story B if: A requires B's database schema changes, B's API endpoints, or B's message contracts to be deployed first
- Mark explicit dependencies in the "Depends On" field
- In the Suggested Sprint Breakdown, ensure dependencies are satisfied within or across sprint boundaries

Common dependency patterns:

- Data layer stories → API stories → Integration stories
- Shared infrastructure (authentication, messaging setup) → feature stories
- Migration stories → cutover stories

### Step 7: Create an Epic

Synthesize an Epic description that encompasses all stories:

- **Epic Title**: "{Feature Name} Modernization"
- **Epic Description**: 2-3 sentences describing the overall scope, target technology stack (e.g., "from JSP + Oracle PLSQL to Spring Boot 4.x + PostgreSQL + Kafka"), and business value
- **Epic Labels**: `modernization`, `{module-name}` (e.g., `modernization`, `order-management`)
- **Link Stories**: Each story is a child of this epic

### Step 8: Legacy File References

For each story, list the legacy source files it replaces or depends on:

- Include the file path and SHA-256 hash from fingerprint.json
- Format: `path/to/file.java` (sha256: `d7a8fbb3...`)
- If fingerprint.json is not available, derive hashes using git or manual calculation
- Include files from all layers: JSPs, controllers, services, DAOs, stored procedures, configuration, tests

Example:

```
**Legacy Source Files**:
- `src/main/webapp/WEB-INF/views/order.jsp` (sha256: `e3b0c44298fc...`)
- `src/main/java/com/acme/service/OrderService.java` (sha256: `d7a8fbb31e...`)
- `src/main/java/com/acme/dao/OrderDAO.java` (sha256: `9f86d0814a...`)
- `src/main/resources/mappers/OrderMapper.xml` (sha256: `abc1234567...`)
- `db/procedures/PROCESS_ORDER.sql` (sha256: `xyz9876543...`)
```

### Step 9: Build Suggested Sprint Breakdown

Create a table showing how stories could be organized into sprints:

```
| Sprint | Stories | Theme | Notes |
|--------|---------|-------|-------|
| 1 | Stories 1, 2, 3 | Data Layer Foundation | Dependencies satisfied; can run in parallel |
| 2 | Stories 4, 5 | API & Service Layer | Depends on Sprint 1; builds on DB schema |
| 3 | Stories 6, 7, 8 | Messaging & Integration | Depends on Sprint 2; parallel work possible |
| 4 | Story 9 | Testing & QA | Full integration testing |
| 5 | Story 10 | Migration & Cutover | Production data migration and switchover |
```

**Heuristics:**

- Sprint 1: foundational stories (data layer, shared infrastructure)
- Middle sprints: feature development (APIs, business logic, integrations)
- Final sprint(s): testing, validation, migration, cutover

### Step 10: Create Legacy File → Story Reverse Index

Build a reverse index showing, for each legacy file, which stories affect it:

```
| Legacy File | Stories | Role | Traceability |
|------------|---------|------|--------------|
| `src/.../OrderService.java` | Story 1, Story 3 | Business Services | FR-001, FR-003, FR-007 |
| `src/.../OrderMapper.java` | Story 2 | Data Access | DB-002, DB-003 |
| `db/procedures/PROCESS_ORDER.sql` | Story 1 | PLSQL | DB-001, TD-005 |
| `src/.../KafkaProducer.java` | Story 6 | Messaging | MSG-001, MSG-002 |
```

This index helps with:
- **Impact analysis** — "If this legacy file changes, which modernization stories are affected?"
- **Validation** — ensuring every legacy file is mapped to at least one story (or explicitly flagged as "not modernized")
- **Migration planning** — understanding which teams own which legacy → modern transitions

### Step 11: Build Traceability Header

Create a header section at the top of jira-stories.md:

```markdown
## Traceability Header

**Generated**: {ISO 8601 timestamp, e.g., 2026-03-22T14:30:00Z}
**Git Commit**: `{commit hash}` (branch: `{branch}`)
**Repo Status**: {clean | dirty, list uncommitted files if dirty}
**Fingerprint Reference**: See `fingerprint.json` for detailed file-level hashes

**Analysis Source**: `modernization-output/{slug}/analysis.md`
**Funcspec Source**: `modernization-output/{slug}/funcspec.md`
**Techspec Source**: `modernization-output/{slug}/techspec.md`

⚠ **Drift Check Required**: After any changes to legacy code, run:
```
/check-drift {slug}
```
to verify that the source files referenced in this document have not diverged since generation.
```

This header allows:
- Downstream tools to locate prerequisite documents
- Git-based traceability ("this story was generated from commit ABC")
- Drift detection (fingerprint comparison)

## Output Template

Create `jira-stories.md` in `modernization-output/{slug}/` with this structure:

```markdown
# JIRA Stories: {Feature Name} Modernization

## Traceability Header
{See Step 11 above}

---

## Epic: {Feature Name} Modernization

**Type**: Epic
**Description**: Modernize {feature} from legacy {stack} to {target stack}.
This epic encompasses {N} stories covering data layer, business logic, APIs,
integrations, and cutover. See funcspec.md for business requirements and
techspec.md for detailed implementation guidance.

**Business Value**: {Why this modernization matters — reduced maintenance burden,
improved scalability, faster feature delivery, etc.}

**Labels**: `modernization`, `{module-name}`, `{priority}`

---

## Story 1: {Title}

**Type**: Story
**Size**: S | M | L | XL
**Description**: {Clear, actionable description of what the story delivers.
Reference the techspec section(s) that provide implementation details.}

**Acceptance Criteria**:
- [ ] {Given/When/Then criteria from funcspec, linked to FR-NNN}
- [ ] {Technical acceptance from techspec (API contract, SQL performance, transaction boundary, etc.)}
- [ ] {Edge case or error handling from behavioral inventory}
- [ ] {Data validation or integrity constraint}
- [ ] {Backward compatibility or migration-specific criterion if applicable}

**Technical Notes**:
This story replaces legacy {component list from analysis.md}. See techspec
section {API-001|MSG-001|DB-001|TD-001} for query/contract/transaction details.
Key constraints: {list any transaction isolation levels, deadlock handling,
data consistency requirements from techspec}.

**Depends On**: {Story N | (none)}

**Functional Spec Refs**: {FR-001, FR-003, FR-007}

**Techspec Refs**: {API-001, DB-002, TD-005}

**Legacy Source Files**:
- `src/main/java/com/acme/service/OrderService.java` (sha256: `d7a8fbb31...`)
- `src/main/java/com/acme/dao/OrderDAO.java` (sha256: `9f86d0814...`)
- `db/procedures/PROCESS_ORDER.sql` (sha256: `abc1234567...`)

**Implementation Hints**:
- {Specific query from techspec that must be preserved}
- {Transaction boundary or isolation level requirement}
- {Integration point that affects other stories}

**Testing Strategy**:
- Unit tests: {number} for business logic, {number} for validation
- Integration tests: {specific scenarios from funcspec acceptance criteria}
- Performance tests: {query/API latency targets from techspec}

---

## Story 2: {Next Story}
{Repeat the structure above for each story}

---

## Suggested Sprint Breakdown

| Sprint | Stories | Theme | Rationale |
|--------|---------|-------|-----------|
| 1 | Stories 1-3 | Data Layer Foundation | Establish database schema and DAOs; enable parallel API development |
| 2 | Stories 4-6 | API & Service Layer | Build REST endpoints and business logic atop sprint 1 foundation |
| 3 | Stories 7-9 | Messaging & Integration | Implement event producers/consumers and external system calls |
| 4 | Stories 10-11 | Testing & QA | End-to-end integration testing and performance validation |
| 5 | Story 12 | Migration & Cutover | Data migration, parallel run validation, production switchover |

---

## Legacy File → Story Mapping

(Reverse index: for each legacy file, which stories are affected)

| Legacy File | Stories | Role | Functional Refs | Techspec Refs |
|------------|---------|------|-----------------|---------------|
| `src/main/java/com/acme/service/OrderService.java` | Story 1, Story 3 | Business Services | FR-001, FR-003, FR-007 | API-001, API-002 |
| `src/main/java/com/acme/dao/OrderDAO.java` | Story 2 | Data Access | FR-002, FR-005 | DB-001, DB-002 |
| `src/main/resources/mappers/OrderMapper.xml` | Story 2 | MyBatis Mapper | FR-002, FR-005 | DB-001, DB-002 |
| `db/procedures/PROCESS_ORDER.sql` | Story 1 | PLSQL Stored Procedure | FR-001, FR-003 | DB-001, TD-005 |
| `src/main/java/com/acme/kafka/OrderEventProducer.java` | Story 6 | Messaging Producer | FR-008, FR-009 | MSG-001, MSG-002 |
| `src/main/webapp/WEB-INF/views/order.jsp` | Story 8 | JSP View | FR-010, FR-011 | API-003 |

---

## Story Dependency Graph

(Optional: ASCII graph or list showing story relationships)

```
Story 1 → Story 3 → Story 5
         ↘ Story 2 ↗
              ↓
          Story 6 → Story 9
           ↙ ↓ ↘
      Story 7 Story 8  Story 10
             ↓
          Story 11 (Cutover)
```

---

## Notes for Refinement

- {Any ambiguities in the specs that should be clarified with product or architecture}
- {Risks or technical debt identified during story creation}
- {Suggested tool choices or frameworks not specified in techspec}
- {Performance optimization opportunities}

```

## Guidelines

1. **Every story must be actionable** — developers should be able to start work immediately with techspec sections referenced explicitly
2. **Acceptance criteria are testable** — Given/When/Then format is verifiable; technical criteria have measurable outcomes (e.g., "query returns in <100ms", "API response validates against schema ABC")
3. **Dual traceability** — every story links to both funcspec (what) and techspec (how) so teams can navigate seamlessly between documents
4. **Complexity reflects techspec detail** — a story referencing a 500-line stored procedure with 15 edge cases is XL, not M
5. **Dependencies are explicit** — no hidden ordering; stories can be parallelized where feasible
6. **Migration and cutover are first-class** — don't treat data migration as an afterthought; plan it as a full story with acceptance criteria
7. **Legacy files have fingerprints** — SHA-256 hashes enable drift detection; if a legacy file changes post-generation, downstream impact is trackable
8. **Sprints respect dependencies** — sprint N should not have stories depending on sprint N+2; front-load foundational work
9. **Reference techspec sections precisely** — "see techspec DB-002" not "see the database section"
10. **Include implementation hints** — developers should find enough concrete guidance in the story to start coding without hunting through three documents

## Output Checklist

- [ ] jira-stories.md created in `modernization-output/{slug}/`
- [ ] Traceability header includes git commit, fingerprint reference, and drift check reminder
- [ ] Epic created with business value description and labels
- [ ] At least one story per major functional area (data, API, messaging, validation, integration, migration, cutover)
- [ ] Each story has: Title, Type, Size (S/M/L/XL), Description, Acceptance Criteria (5-10 items), Depends On, FR/Techspec refs, Legacy files with SHA-256 hashes
- [ ] Acceptance criteria follow Given/When/Then format and reference funcspec FRs
- [ ] Technical notes explicitly reference techspec sections (API-NNN, DB-NNN, MSG-NNN, TD-NNN)
- [ ] Legacy source files listed with SHA-256 hashes from fingerprint.json
- [ ] Sprint breakdown table created with rationale
- [ ] Legacy File → Story reverse index created and complete
- [ ] All dependencies are documented and cycle-free (DAG)
- [ ] Every story is estimated (S/M/L/XL) and justified
- [ ] Migration/cutover stories are included and fully detailed
- [ ] All FRs from funcspec are mapped to at least one story
- [ ] All techspec sections are referenced by at least one story (or flagged as optional/deferred)

## Implementation Notes

1. **Slug discovery** — before starting, verify `modernization-output/{slug}/` contains analysis.md, funcspec.md, and techspec.md
2. **Fingerprint handling** — if fingerprint.json exists, use it for file hashes; otherwise compute SHA-256 using git or file content
3. **Git integration** — capture current git commit and branch at generation time; include in traceability header
4. **FR indexing** — build a lookup table of FR-NNN → {story list} to verify all FRs are covered
5. **Techspec section extraction** — scan techspec.md for section headers matching pattern `^\s*(API|MSG|DB|TD)-\d+` and index them
6. **Complexity justification** — for each estimate (S/M/L/XL), document the reasoning (e.g., "XL: 500-line stored procedure with 15 edge cases + 3 integration points")
7. **Sprint sequencing** — use dependency analysis to ensure sprint 1 can run independently, sprint 2 depends only on sprint 1, etc.
8. **Cutover story detail** — migration/cutover stories should be as detailed as feature stories; include data validation, rollback procedures, and parallel run strategy

---

**Related Skills**: `/analyze-entry`, `/gen-funcspec`, `/gen-techspec`, `/modernize`

**Next Steps**: After JIRA stories are reviewed and prioritized, use `/gen-code {slug} {service-name}` to generate Spring Boot 4.x source code.
