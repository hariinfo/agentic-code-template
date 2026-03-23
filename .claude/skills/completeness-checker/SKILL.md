---
name: verify-completeness
description: Verify modernized code completeness against legacy system through multi-pass behavioral, structural, and requirement traceability analysis. Detects discovery gaps, translation gaps, implementation gaps, and unintended scope creep. Trigger on: verify, completeness, coverage, missed, gap analysis, behavioral coverage, nothing missed, legacy coverage. Produces completeness-report.md with traceability matrix, gap prioritization, and actionable remediation steps.
---

# Completeness Verification Skill (verify-completeness)

## Overview

After modernized code has been written (either manually or generated), this skill performs comprehensive verification that the new code covers every behavior, business rule, integration, code path, and side effect from the legacy system.

Modernization failures typically stem from three gap types:
1. **Discovery gaps** — the initial analysis missed a legacy component entirely (hidden stored procedure, undocumented scheduled job, configuration-driven behavior)
2. **Translation gaps** — the analysis found it and funcspec described it, but the JIRA story didn't properly capture it (or captured it incorrectly)
3. **Implementation gaps** — the story was correct but modernized code doesn't fully implement it (missed an edge case, forgot error handling, dropped a side effect)

This skill catches all three by working from ground truth—the actual legacy code—and checking actual modernized code against it. It also detects unintended scope creep (modern code with no legacy equivalent).

## Trigger

```
/verify-completeness {slug} {path-to-modernized-code}
```

**Parameters:**
- `{slug}` — directory name under `modernization-output/` containing analysis artifacts
- `{path-to-modernized-code}` — absolute or relative path to the modernized codebase root

**When to Run:**
- After implementing some or all JIRA stories to validate progress
- As a final gate before decommissioning legacy code
- When stakeholders ask "did we implement everything?"
- To identify what work remains

## Inputs

The skill reads from:

1. **`modernization-output/{slug}/analysis.md`** — Behavioral Inventory with behavior IDs (B-001, B-002, ...), descriptions, and legacy source locations
2. **`modernization-output/{slug}/fingerprint.json`** — File list, signatures, and public methods/endpoints/SQL/handlers per file
3. **`modernization-output/{slug}/funcspec.md`** — Functional requirements (FR-001, FR-002, ...) with acceptance criteria and source traceability
4. **`modernization-output/{slug}/jira-stories.md`** — JIRA stories with legacy file mappings and acceptance criteria
5. **Legacy codebase** — original source files at their repository locations
6. **Modernized codebase** — new source files at the provided path

## Verification Logic — Five-Pass Approach

### Pass 1: Behavioral Coverage (Legacy → Modern)

For each entry in the Behavioral Inventory:

1. Read the legacy source at the cited location to confirm behavior exists
2. Extract the essential business logic (not implementation details)
3. Search the modernized codebase for equivalent implementation:
   - Look for same business logic, possibly expressed differently
   - Check language idioms (e.g., Java exception handling → TypeScript try/catch)
   - Check architectural shifts (e.g., stored procedure → microservice endpoint)
4. Classify each behavior:
   - **🟢 COVERED** — modernized code implements the behavior exactly
   - **🟡 PARTIAL** — modernized code implements it but incompletely (e.g., missing error case, timeout incorrect)
   - **🔴 MISSING** — not found in modernized code
   - **⏭️ INTENTIONALLY SKIPPED** — documented as out of scope (reference the decision or story)
5. For COVERED and PARTIAL items, document modern location and any differences

### Pass 2: Structural Coverage (Files → Modern)

For each file in `fingerprint.json`:

1. Extract all public methods, REST endpoints, SQL operations, message handlers, scheduled jobs
2. For each discovered API/method/handler:
   - Search modernized codebase for a counterpart (same name, signature, or equivalent functionality)
   - Check if it's deprecated or moved
3. Flag any legacy method/endpoint with no modern equivalent
4. Note methods that are modernized but with different signatures (parameters, return types, exceptions)

### Pass 3: Requirement Traceability (Funcspec → Stories → Code)

For each functional requirement (FR-001, FR-002, ...):

1. Verify it maps to at least one JIRA story (check `jira-stories.md`)
2. For each story, verify it maps to actual modernized code (file, class, method names)
3. Check that acceptance criteria from the story are testable against the new code:
   - Are Given/When/Then assertions implementable?
   - Do test assertions match code implementation?
4. Flag broken traceability:
   - FR with no story (translation gap)
   - Story with no code (implementation gap)
   - Acceptance criteria that don't match code behavior

### Pass 4: Reverse Check (Modern → Legacy)

Search the modernized codebase for logic with NO corresponding legacy behavior:

1. Scan for methods/endpoints/handlers that weren't in fingerprint.json
2. For each new piece of logic, ask: "Does this exist in legacy code?"
3. If not, classify:
   - **Intentional enhancement** — documented new feature, approved by stakeholders
   - **Hallucinated requirement** — code exists but no legacy equivalent and not approved (risk)
   - **Framework boilerplate** — necessary infrastructure code, not business logic
4. Flag anything unexplained as potential scope creep

### Pass 5: Re-scan Legacy for Missed Behaviors

Re-read each legacy source file looking for behaviors NOT in the Behavioral Inventory:

1. Hidden conditional branches (nested if-else, switch cases, ternary chains)
2. Catch blocks with business logic (not just logging, but actual recovery or side effects)
3. Default cases and fallback behaviors
4. Configuration-driven behavior (properties files, feature flags, ENV variables)
5. Implicit behaviors (framework conventions, annotation side effects, listener patterns)
6. Hidden dependencies (static initializers, lazy loading, circular references)
7. Timing or ordering assumptions (database sequence usage, cache invalidation)
8. Resource cleanup (try-finally blocks, resource leaks)

If new behaviors are found, classify as **Discovery Gaps (DG-001, DG-002, ...)** and flag as potentially missed by original analysis.

## Output — `completeness-report.md`

Create this file in the `modernization-output/{slug}/` directory.

### Structure

```markdown
# Completeness Report: {slug}

**Generated**: {ISO timestamp}
**Modernization Initiative**: {slug}
**Legacy Codebase Root**: {provided path or "original repo"}
**Modernized Codebase Root**: {path-to-modernized-code}

---

## Summary

- **Behaviors in Legacy Inventory**: {count from Behavioral Inventory}
- **Behaviors covered in modern code**: {count} ({percentage}%)
- **Behaviors partially covered**: {count} ({percentage}%)
- **Behaviors missing**: {count} ({percentage}%)
- **Behaviors intentionally skipped**: {count}
- **Discovery gaps (missed by analysis)**: {count}
- **Unexplained new behaviors in modern code**: {count}
- **Functional requirements with full traceability**: {count} ({percentage}%)

---

## Overall Completeness Indicator

**Completeness Score**: {percentage}%

**Color-Coded Status**:
- 🟢 **95%+** — Excellent. Ready for decommissioning legacy code.
- 🟡 **80-95%** — Good. Minor gaps; identify and prioritize before go-live.
- 🟠 **60-80%** — Concerning. Significant gaps; recommend re-analysis or scope review.
- 🔴 **<60%** — Critical. Substantial behaviors missing; do not deploy.

**Risk Assessment**:
- {List severity: HIGH (user-facing functions), MEDIUM (operational/integration), LOW (internal, non-critical)}

---

## Traceability Matrix

| B-ID | Legacy Behavior | Legacy Location | Funcspec Ref | Story Ref | Modern Location | Status | Notes |
|------|-----------------|-----------------|--------------|-----------|-----------------|--------|-------|
| B-001 | Validate order total > 0 | OrderService.java:142 | FR-003 | STORY-42 | NewOrderService.ts:88 | 🟢 COVERED | — |
| B-002 | Send Kafka event on creation | OrderService.java:198 | FR-007 | STORY-51 | OrderEventPublisher.ts:34 | 🟢 COVERED | Now uses CloudEvents format |
| B-003 | Return 403 if user lacks ROLE_ADMIN | OrderController.java:55 | FR-012 | STORY-48 | middleware/auth.ts:22 | 🟢 COVERED | — |
| B-004 | Retry DB insert 3x on deadlock | OrderDAO.java:88 | FR-003 | STORY-42 | — | 🔴 MISSING | No retry logic found; **ACTION**: implement in modern code |
| B-005 | Cache order lookup for 5 min TTL | OrderService.java:67 | — | — | CacheLayer.ts:156 | 🟡 PARTIAL | Caching exists but TTL is 15min, not 5min; **ACTION**: verify TTL is still correct business rule |
| B-006 | Log all approvals to AUDIT_LOG table | ApprovalDAO.java:201 | — | — | — | ⏭️ INTENTIONALLY SKIPPED | Out of scope per STORY-52; compliance team approved migration to audit service |
| ... | ... | ... | ... | ... | ... | ... | ... |

---

## Discovery Gaps (Behaviors Missed by Original Analysis)

| Gap ID | Behavior Found | Legacy Location | Impact | Recommendation |
|--------|----------------|-----------------|--------|-----------------|
| DG-001 | Write audit log to AUDIT_LOG table on every order update | OrderDAO.java:201 | HIGH — compliance/regulatory | Add to funcspec, create new story, implement in modern code |
| DG-002 | Feature flag ORDER_V2_ENABLED gates new validation path | OrderService.java:150 | MEDIUM — operational behavior | Decide if modern code needs equivalent feature toggle; if yes, implement; if no, document removal |
| DG-003 | Deadletter queue processing for failed Kafka messages | OrderConsumer.java:245 (catch block) | MEDIUM — reliability | Modern code may use platform DLQ; confirm equivalent handling exists |
| ... | ... | ... | ... | ... |

**How to Proceed:**
1. For HIGH-impact gaps, create JIRA story and implement immediately
2. For MEDIUM gaps, review with product owner and architect
3. For LOW gaps, document decision to remove or defer

---

## Structural Coverage

### Legacy Methods/Endpoints Without Modern Equivalent

| Legacy File | Method/Endpoint | Purpose | Signature | Risk | Recommendation |
|------------|----------------|---------|-----------|------|-----------------|
| OrderService.java | cancelOrder(Long id) | Cancels order and reverses inventory | public void cancelOrder(Long orderId) throws ServiceException | HIGH — user-facing | Implement immediately or document removal |
| OrderDAO.java | findOrdersByStatus(String status) | Query by order status | public List<Order> findOrdersByStatus(String status) | MEDIUM — reporting/internal | Implement or confirm query moved elsewhere |
| ... | ... | ... | ... | ... | ... |

### Modern Code Without Legacy Equivalent

| Modern File | Method/Endpoint | Type | Explanation Needed | Risk |
|------------|----------------|------|-------------------|------|
| NewOrderService.ts | archiveOrder() | Method | Not found in legacy codebase | MEDIUM — is this intentional new scope? Confirm with product owner |
| OrderEventPublisher.ts | publishMetricsEvent() | Handler | No equivalent in legacy Kafka; appears to be new observability feature | LOW — enhancement, not gap |
| ... | ... | ... | ... | ... |

---

## Requirement Traceability Gaps

| Funcspec Req | Description | Has Story? | Has Modern Code? | Gap Type | Action |
|-------------|-------------|-----------|-----------------|----------|--------|
| FR-008 | System must validate order address matches shipping carrier zones | ❌ No | — | Translation gap | Create story and implement, or remove requirement |
| FR-011 | System must notify customer on approval | ✅ STORY-67 | ❌ No | Implementation gap | Implement notification logic in modern code |
| FR-015 | Handle concurrent order creation (idempotent) | ✅ STORY-71 | ✅ Yes | — | ✅ Verified complete |
| ... | ... | ... | ... | ... | ... |

---

## Recommendations & Prioritized Actions

### Critical (Block Go-Live)

1. **Implement B-004 — Retry logic for deadlock handling**
   - Risk: Data loss or duplicate entries on transient DB contention
   - Effort: 2-3 story points
   - Reference: STORY-42 (already created, mark as blocked)
   - Legacy source: OrderDAO.java:88-110
   - Modern target: Data access layer retry handler

2. **Create story for DG-001 — Audit logging to AUDIT_LOG table**
   - Risk: Compliance and forensics gap
   - Effort: 3-5 story points
   - Legacy source: OrderDAO.java:195-210
   - Action: New JIRA story, implement in modern DAL

### High Priority (Before Go-Live)

3. **Verify FR-008 requirement with product owner**
   - Action: Schedule clarification meeting
   - Outcome: Decide if requirement is still valid; if yes, create story

4. **Confirm B-005 TTL is still correct (5 min vs. 15 min)**
   - Action: Review with original developer
   - Outcome: Update modern code to match business requirement or get approval to change

### Medium Priority (Within 30 Days)

5. **Review modern code scope creep — archiveOrder() in NewOrderService.ts**
   - Action: Verify with product owner that this is intentional enhancement
   - Outcome: Document decision and link to original feature request

6. **Re-run /analyze-entry to capture newly discovered behaviors (DG-001, DG-002, DG-003)**
   - Action: Re-analyze legacy entry points with focus on config-driven and implicit behaviors
   - Outcome: Update Behavioral Inventory and funcspec

---

## Conclusion

**Overall Assessment**: {State clearly whether this modernization is feature-complete, what work remains, and whether it's safe to proceed}

**Next Steps**:
1. {Action}
2. {Action}
3. {Action}

**Sign-Off Needed From**:
- [ ] Architect (confirms technical completeness)
- [ ] Product Owner (confirms business requirements met)
- [ ] QA Lead (confirms test coverage for all behaviors)
- [ ] Platform Team (confirms infrastructure/dependency completeness)

---

## Appendix: Search Queries Used

*Document all code searches performed, keywords used, and patterns matched to aid reproducibility and future audits.*

| Search Term | Pattern | Files Scanned | Matches Found | Notes |
|------------|---------|----------------|----------------|-------|
| {search} | {regex or literal} | {count} | {count} | {e.g., "Found in OrderService and NewOrderService"} |

```

---

## Guidelines for the Skill

1. **Be Exhaustive** — Don't assume behaviors are covered. Actively search; confirm absence before marking MISSING.

2. **Distinguish Implementation from Behavior** — A behavior can be implemented many ways (Oracle PLSQL → TypeScript, sync → async, single thread → parallel). Look for equivalent outcome, not identical code.

3. **Trace Every Gap** — For each gap found, document:
   - What legacy code showed it (file, line, snippet)
   - Where modern code should have it (file, class, method name predicted)
   - Why it's missing (oversight, design decision, intentional removal)

4. **Flag Implicit Requirements** — Configuration, feature flags, environment-specific behavior, framework conventions. These are often missed in initial analysis.

5. **Color Coding is Not Optional** — Use 🟢 🟡 🟠 🔴 consistently so stakeholders can skim severity at a glance.

6. **Prioritize by Risk** — HIGH-risk gaps (user-facing, data integrity, compliance) first. LOW-risk (performance tweaks, internal logic) can be deferred.

7. **Reverse-Check is Critical** — Unexplained new code is a red flag. Hallucinated requirements waste effort and complicate testing.

8. **Reference Everything** — Every gap, every behavior, every discrepancy must link back to legacy source code and/or story number. This enables developers to investigate and fix.

9. **Suggest Tests** — For each missing behavior, suggest a test case (Gherkin or unit test format) that would verify it once implemented.

10. **Be Actionable** — Don't just list gaps. Recommend what to do: implement, remove, defer, clarify with stakeholder. Include effort estimates if possible.

## Output Checklist

- [ ] `completeness-report.md` created in `modernization-output/{slug}/`
- [ ] Summary section includes counts and percentages for all gap types
- [ ] Overall Completeness Indicator (color-coded status) is clear
- [ ] Traceability Matrix has at least one row per 5 behaviors in Behavioral Inventory
- [ ] Discovery Gaps section identifies at least 3 potential missed behaviors (or states "none found")
- [ ] Structural Coverage lists all legacy public methods and their modern equivalents (or notes absence)
- [ ] Requirement Traceability Gaps section verifies FR → Story → Code chain
- [ ] Reverse-check identifies any modern code without legacy equivalent
- [ ] Recommendations are prioritized (Critical, High, Medium) and actionable
- [ ] All gaps include legacy source location (file:line) and modern target location (if known)
- [ ] Sign-off section identifies who must approve before go-live
