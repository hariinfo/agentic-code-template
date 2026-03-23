---
name: funcspec-writer
description: Generate technology-agnostic functional specification and requirements document from legacy analysis, extracting business rules and acceptance criteria to guide modernization. Produces funcspec.md with actors, functional requirements, data model, integration points, and business-critical acceptance criteria for product owners and architects.
---

# Functional Specification Writer (gen-funcspec)

## Overview

This skill translates the raw technical analysis from a legacy entry point into a **functional specification** — a business-focused requirements document that describes *what* the feature does, independent of how the legacy code implements it.

The funcspec serves as the bridge between analysis (low-level component inventory) and technical specification (implementation details). It extracts implicit business rules from the legacy code, identifies data entities and their contracts, and flags edge cases that modernization must preserve.

## Inputs

You will receive a **slug** — the directory name under `modernization-output/` containing:
- `analysis.md` — component inventory, behavioral inventory, data flow diagram
- `source-extracts.md` — raw technical details extracted per-file (SQL, API schemas, error handling, config, messaging, security, transactions)

## Process

### 1. Read and Index the Analysis

Load `analysis.md` and extract:
- **Component inventory** — all classes, JSPs, stored procedures, filters, listeners discovered
- **Behavioral inventory** — entry point flow, call sequences, request/response paths
- **Data flow** — how data moves from user input → service layer → database → output
- **Identified business rules** — any rules explicitly mentioned during analysis

### 2. Read Source Extracts and Extract Business Rules

Load `source-extracts.md` and scan each file section for:

**From API/Controller sections:**
- Request/response schemas
- Validation rules (required fields, patterns, ranges)
- Authorization checks and role requirements
- HTTP status codes and error responses
- Request/response headers and metadata

**From Service/Business Logic sections:**
- Conditional logic that implements business rules (if-else blocks, state machines)
- Threshold values, flags, and configuration-driven decisions
- Rules about state transitions (order status flow, approval workflows)
- Cascading operations (what happens when one action triggers others)
- Calculation logic (pricing, discounts, fees)

**From DAO/Database sections:**
- Table structure, columns, constraints
- Primary keys, foreign keys, and relationships
- Unique constraints and validation rules
- Views or derived data that systems depend on
- Soft-delete patterns or archival logic

**From SQL/PLSQL sections:**
- Stored procedure logic — what preconditions must be met before execution
- Transaction boundaries and isolation levels
- Lock modes and deadlock handling
- Sequence usage (ordering of operations)
- Trigger logic that enforces invariants

**From Messaging sections:**
- Kafka topics, consumer groups, and DLQ handling
- Message schemas and contract expectations
- Ordering guarantees and delivery semantics
- When events are fired and in what sequence

**From Configuration sections:**
- Feature flags and configuration-driven behavior
- Property-based rules (approval thresholds, rate limits, timeouts)
- Environment-specific variations

**From Security sections:**
- Authentication requirements
- Role and permission requirements per operation
- CSRF, XSS, injection prevention mechanisms
- Data sensitivity classifications

**From Error Handling sections:**
- What exceptions are caught and how they're handled
- Retry logic and backoff strategies
- Fallback behaviors
- Degraded-mode operations

### 3. Separate Concerns: What vs. How

Translate legacy implementation details into business-agnostic requirements:

- **Technology-specific details → abstracted** (e.g., "Oracle PLSQL stored procedure updates ORDER_HISTORY table" becomes "System maintains audit trail of order status changes")
- **Implementation patterns → business semantics** (e.g., "Exponential backoff up to 3 retries on DeadlockLoserDataAccessException" becomes "System handles transient contention gracefully")
- **Framework concepts → user actions** (e.g., "@Transactional(isolation=SERIALIZABLE)" becomes "Payment processing must not be interrupted by concurrent updates to the same order")

### 4. Build Functional Requirements with Acceptance Criteria

For each business rule discovered, create a functional requirement (FR) in Given/When/Then (Gherkin) format:

**FR-NNN: [Requirement Title]**
- **Description**: What the feature does in plain English
- **Business Rule**: Why this matters or what invariant it protects
- **Acceptance Criteria**:
  - Given [precondition], When [action], Then [outcome]
  - (repeat for all scenarios — happy path, error cases, edge cases)
- **Source**: [which legacy component this was derived from, e.g., "OrderService.java:78", "order-approval-proc.sql:§Create Pending Approval"]

### 5. Identify Data Entities and Relationships

From the tables, stored procedures, and API schemas in source-extracts.md, build a data model:

- List each entity (table, domain object, aggregate root)
- For each entity, list attributes, types, nullability, constraints
- Document relationships (1:1, 1:N, M:N)
- Highlight foreign keys and referential integrity
- Note any legacy denormalization and why it exists
- Call out audit columns (created_at, updated_at, created_by)

### 6. Identify Integration Points

From the Messaging, API, and Configuration sections of source-extracts.md:

- **External APIs** — which systems this feature calls, and which call it
- **Kafka topics** — producers and consumers, message schemas
- **Shared databases** — tables this feature reads/writes, and which other systems depend on those tables
- **Configuration systems** — where runtime behavior is controlled
- **Authentication/authorization systems** — how users are authenticated and what permissions are enforced
- **File systems or blob storage** — if applicable
- **Caching layers** — shared caches that multiple components depend on

### 7. Flag Edge Cases and Error Handling

From error handling sections in source-extracts.md, extract:

- What goes wrong and how the legacy code recovers
- Validation errors and what triggers them
- State conflicts (e.g., "double-click on submit button during processing")
- Concurrent operation issues (race conditions, deadlocks)
- Resource exhaustion (quota limits, capacity constraints)
- Timeouts and retry limits
- Data consistency edge cases (orphaned records, missing relationships)

### 8. Flag Modernization Opportunities

Review the legacy implementation for areas where modernization could improve:

- **Performance** (e.g., N+1 queries, missing indexes, cache inefficiency)
- **User experience** (e.g., long-running operations that could be async)
- **Maintainability** (e.g., tangled dependencies, implicit contracts)
- **Scalability** (e.g., shared state, locking contention)
- **Testing** (e.g., hard to test legacy patterns)

## Output Template

Create `funcspec.md` in the slug directory with this structure:

```markdown
# Functional Specification: {Feature Name}

**Status**: Draft
**Modernization Initiative**: {slug}
**Last Updated**: {today's date}
**Derived From**: {entry point path}

---

## 1. Overview

{2-3 sentences describing what this feature does from a business perspective — don't mention legacy technology}

**Key Business Value**: {Why this feature exists}

**Scope Boundaries**:
- Includes: {what is in scope}
- Excludes: {what is explicitly not in scope}

---

## 2. Actors & Personas

| Actor | Context | Permissions | Frequency |
|-------|---------|-------------|-----------|
| {Actor Name} | {What they're trying to do} | {What they can and cannot do} | {How often they use this} |

{Repeat for each actor}

**Interaction Patterns**: {How these actors interact with each other and the system}

---

## 3. Functional Requirements

{Organize FRs into logical groups if there are many (e.g., "Create Order", "Approve Order", "Error Handling")}

### FR-001: {Requirement Title}

**Description**: {What happens in plain English}

**Business Rule**: {Why this rule exists; what invariant it protects}

**Acceptance Criteria**:
- Given {precondition}, When {action}, Then {expected outcome}
- Given {precondition}, When {action with error}, Then {error handling}
- Given {edge case}, When {action}, Then {behavior}

**Source**: {Legacy component(s) — e.g., "OrderService.createOrder():45-120; OrderDAO.insertOrder():78"}

**Related FRs**: {Any other FRs that depend on or conflict with this one}

---

### FR-002: {Next Requirement}

{Repeat the structure above}

---

## 4. Data Model

**Entity Relationship Diagram** (textual description or ASCII diagram):

```
{Entity Names and their relationships}
```

### Entity: {Entity Name}

**Description**: {What this entity represents}

**Attributes**:
| Attribute | Type | Nullable? | Constraints | Purpose |
|-----------|------|-----------|-------------|---------|
| {name} | {type} | {Y/N} | {PK, FK, UNIQUE, etc.} | {Why this exists} |

**Relationships**:
- {name} has many {related entity} (1:N)
- {name} belongs to {related entity} (N:1)

**Key Constraints**:
- Primary Key: {column(s)}
- Foreign Keys: {references}
- Unique Constraints: {columns}

**Legacy Notes**: {Any denormalization, views, or workarounds}

{Repeat for each entity}

---

## 5. Integration Points

| System | Direction | Protocol | Frequency | Purpose | Contract |
|--------|-----------|----------|-----------|---------|----------|
| {System} | In/Out/Both | {REST, Kafka, DB, etc.} | {When/how often?} | {What business value?} | {ref to source-extracts.md} |

{Repeat for each integration}

### Kafka Topics

| Topic | Producer | Consumer(s) | Message Schema | Ordering | DLQ |
|-------|----------|-------------|----------------|----------|-----|
| {topic} | {source} | {destination(s)} | {fields} | {guarantee} | {if yes, which} |

### Shared Database Tables

| Table | Read By | Write By | Critical Columns | Constraints | Migration Path |
|-------|---------|----------|------------------|-------------|-----------------|
| {table} | {consumers} | {producers} | {important cols} | {business rules} | {modernization notes} |

---

## 6. Non-Functional Considerations

### Performance

- **Expected Throughput**: {X requests/sec, Y messages/sec}
- **Latency Target**: {P95 latency, max acceptable}
- **Scaling**: {Horizontal? Vertical? Stateless?}
- **Data Volume**: {Record counts, table sizes if known}
- **Batch vs. Real-Time**: {Workload characteristics}

### Security

- **Authentication**: {How are users authenticated?}
- **Authorization**: {Role-based, permission-based, attribute-based?}
- **Data Sensitivity**: {PII, financial data, compliance boundaries}
- **Audit Requirements**: {What must be logged?}
- **Encryption**: {In transit? At rest?}
- **Compliance**: {GDPR, PCI, SOX, etc.?}

### Reliability & Resilience

- **SLA/SLO**: {Availability target, error budgets}
- **Retry Strategy**: {How are transient failures handled?}
- **Circuit Breaker**: {Is this needed for external calls?}
- **Fallback**: {What happens when a dependency fails?}
- **Monitoring**: {Critical metrics to track}

### Audit & Compliance

- **Change Audit**: {What changes must be logged? Who changed it, when, what was old/new?}
- **Access Audit**: {Who accessed what data, when?}
- **Data Retention**: {How long must records be kept?}
- **Regulatory**: {Any specific compliance rules?}

---

## 7. Migration Notes

### Data Migration Considerations

- {Any data cleanup or transformation needed to move to modernized version?}
- {Legacy data that might not fit the new schema?}
- {Referential integrity issues to resolve?}

### Feature Parity Gaps

- {Behaviors in legacy code that are undocumented or implicit}
- {Edge cases that must be preserved}
- {Workarounds in legacy code — is there a better way?}

### Suggested Phasing

**Phase 1**: {What can be modernized first, independently}
**Phase 2**: {What depends on Phase 1 being complete}
**Phase 3**: {Final integration}

**Backward Compatibility**: {How long must we support both old and new? API versioning strategy?}

---

## 8. Open Questions

- {Any ambiguity in the legacy code that needs stakeholder clarification?}
- {Business rules that seem contradictory or unclear?}
- {Unused features that might be dead code?}
- {Constraints that might be outdated?}

**Next Steps**: {Who should answer these questions, and by when?}

---

## Traceability

| FR ID | Analysis Section | Source File | Line Range |
|-------|-----------------|-------------|------------|
| FR-001 | {section} | {file} | {lines} |
| FR-002 | {section} | {file} | {lines} |

---

## Sign-Off

- [ ] Product Owner reviewed and approved
- [ ] Architect reviewed for modernization feasibility
- [ ] Data team reviewed data model
- [ ] Security team reviewed security requirements

**Approval Date**: TBD
```

## Guidelines

1. **Be Pushy About Business Value** — emphasize what the feature *does* and why, not how the legacy code does it
2. **Use Gherkin for Acceptance Criteria** — Given/When/Then format is language-agnostic and testable
3. **Trace Every FR Back to Source** — when modernized code is written, developers must be able to find the original requirement
4. **Separate Concerns** — don't write "Oracle PLSQL calls stored procedure order_approval_proc". Write "System applies business approval rules before confirming order"
5. **Flag Implicit Rules** — if the legacy code silently handles cases, make them explicit in acceptance criteria
6. **Call Out Integration Dependencies** — downstream systems depend on your contracts; document them
7. **Don't Assume Technology** — a functional spec should be implementable in Java, Python, Go, or any modern language

## Output Checklist

- [ ] funcspec.md created in `modernization-output/{slug}/`
- [ ] All actors and their roles are documented
- [ ] At least 3-5 functional requirements per major feature area
- [ ] Each FR has acceptance criteria in Given/When/Then format
- [ ] Each FR traces back to legacy source code
- [ ] Data entities and relationships are documented
- [ ] All integration points are identified (APIs, Kafka, shared tables, etc.)
- [ ] Non-functional requirements (performance, security, audit) are explicit
- [ ] Migration path is sketched (phasing, backward compatibility)
- [ ] Open questions are listed for stakeholder clarification
