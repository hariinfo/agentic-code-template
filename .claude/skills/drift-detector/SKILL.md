---
name: check-drift
description: Detect and classify source code changes since analysis snapshot. Compares current codebase against fingerprint.json baseline to identify file deletions, modifications, new files, and signature changes. Cross-references drift against JIRA story mappings to classify impact (CRITICAL/SIGNIFICANT/MINOR/NEW SCOPE) and surfaces stories needing revision before implementation.
---

# Drift Detector (check-drift)

## Overview

This skill compares the current state of legacy source files against the `fingerprint.json` snapshot to detect what has changed since modernization artifacts were generated. It produces an actionable **drift report** that tells you which JIRA stories need attention, which files have been modified, deleted, or unexpectedly added, and the business impact of those changes.

The drift detector helps teams answer critical questions:
- **Has the legacy code I'm supposed to modernize changed since we analyzed it?**
- **Do my JIRA stories still reflect reality, or has the codebase drifted?**
- **Which stories are at risk due to unexpected changes?**
- **Are there new components in scope that we missed?**

## Trigger

`/check-drift <slug>`

Run this:
- After a sprint when code may have changed
- Before starting implementation of a modernization story
- As a CI/CD check before deployment
- On a schedule to monitor drift over time
- After major refactoring in the legacy codebase

## Inputs

You will receive a **slug** — the directory name under `modernization-output/` containing:

- `fingerprint.json` (required) — baseline commit hash, file list, per-file SHA-256 hashes, and key_signatures
- `jira-stories.md` (required) — functional stories with file→story reverse mapping
- The current legacy codebase on disk (at the project root)
- The git repository (accessed via `git diff`, `git log`, `git show`)

## Process

### 1. Read and Validate Fingerprint

Load `modernization-output/{slug}/fingerprint.json` and extract:
- **baseline_commit** — the git commit hash when the analysis was performed
- **baseline_branch** — the branch (usually main/master)
- **baseline_timestamp** — when the snapshot was taken
- **files** — array of file objects, each containing:
  - `path` — relative path from project root
  - `sha256` — hash of the file content at baseline
  - `key_signatures` — array of structured signatures (method signatures, SQL DDL, etc.)
  - `file_type` — (java, jsp, sql, plsql, kafka, etc.)
  - `size_bytes` — file size at baseline

Validate that the baseline commit exists and is reachable:
```bash
git rev-parse {baseline_commit}
```

If the baseline commit is unreachable, report this as a CRITICAL issue and halt (cannot compare against a missing baseline).

### 2. Analyze Each Baseline File

For each file in the fingerprint, perform sequential checks:

#### 2a. Check Existence
```bash
if [ ! -f "{filepath}" ]; then
  # File was deleted
  classify as CRITICAL drift
fi
```

Deleted files always classify as CRITICAL because stories may depend on components they contained.

#### 2b. Compute Current Hash and Compare
```bash
current_sha256 = sha256sum({filepath})
if (current_sha256 != baseline_sha256) {
  // File was modified
  git_diff_output = git diff {baseline_commit} HEAD -- {filepath}
} else {
  // File unchanged
  continue to next file
}
```

#### 2c. Extract Changes from Git Diff

Parse the `git diff` output to determine:
- Lines added, removed, modified
- Methods/classes that appear in diff hunks (using simple regex patterns)
- SQL statements that changed
- Configuration keys that changed

#### 2d. Compare Key Signatures

Extract current key_signatures from the modified file (scan for Java method signatures, SQL DDL, etc.) and compare against the stored baseline signatures:

**For Java files:**
- Scan for `public`, `protected`, `private` method signatures
- Look for class definitions (`public class X`, `interface Y`)
- Note parameter changes, return type changes, exception declarations

**For SQL/PLSQL files:**
- Scan for `CREATE PROCEDURE`, `CREATE FUNCTION`, `CREATE TABLE`, `ALTER TABLE`
- Look for column additions, deletions, type changes
- Note index or constraint modifications

**For JSP files:**
- Track request parameter names used in `request.getParameter()`
- Note scriptlet variable definitions

**For Kafka files:**
- Track topic names referenced in `KafkaTemplate`, `@KafkaListener`
- Note consumer group IDs

Signature comparison:
```
added_signatures = current_signatures - baseline_signatures
removed_signatures = baseline_signatures - current_signatures
modified_signatures = (found in both but with changes)
```

#### 2e. Get Commit History

For each changed file, retrieve commit messages explaining *why* it changed:
```bash
git log {baseline_commit}..HEAD --oneline -- {filepath}
```

Extract commit hashes and messages for use in the drift report.

### 3. Detect New Files in Same Scope

Search the codebase for files in the same packages/directories as analyzed files but not present in the fingerprint:

**For Java:**
- If `src/main/java/com/acme/service/OrderService.java` was analyzed, search for other `.java` files in `src/main/java/com/acme/service/` and `src/main/java/com/acme/dao/`, etc.
- Check if new ServiceImpl classes, new DAOs, new filters appeared

**For SQL/PLSQL:**
- If `db/procedures/order_approval_proc.sql` was analyzed, search `db/procedures/` for new `.sql` files
- Check `db/schema/` for new table definitions

**For Kafka:**
- If `/src/main/java/com/acme/kafka/OrderEventConsumer.java` was analyzed, search `/src/main/java/com/acme/kafka/` for new consumer or producer classes

New files in the same scope are classified as **NEW SCOPE** drift and flagged for evaluation.

### 4. Cross-Reference Against Story Mappings

Load `modernization-output/{slug}/jira-stories.md` and extract the **Legacy File → Story Mapping** section (usually a table mapping file paths to JIRA story IDs).

For each changed/deleted file:
1. Look up which stories reference it
2. Note those stories as potentially affected
3. Evaluate the severity of the change

### 5. Classify by Impact Level

For each changed or deleted file, assign an impact classification:

#### **CRITICAL** — Breaks story assumptions
Changes that fundamentally invalidate story acceptance criteria:
- **File deleted** — if the story depends on components in that file (e.g., a DAO, service class)
- **Method removed** — if a story's acceptance criteria calls out a specific method signature
- **Method signature changed** (parameters added, removed, or type changed) — stories may have assumptions about the old signature
- **Table schema change** (columns removed, type changed, constraint added) — DAO/SQL queries may break
- **API contract broken** (response schema altered, endpoint removed) — dependent stories may expect old contract
- **Kafka topic removed or renamed** — consumers/producers expecting old topic name will fail
- **Key stored procedure removed or signature changed** — batch jobs or workflows depending on it will fail

Action needed: **Story must be reviewed and updated before implementation begins.**

#### **SIGNIFICANT** — Alters business logic, stories may still be valid
Changes that modify behavior but stories might adapt:
- **Business logic modified** (validation rules changed, calculation modified) — story acceptance criteria may need review
- **New error handling added** — may open new edge cases
- **SQL query logic changed** (WHERE clause modified, JOINs added) — data returned may differ
- **New conditional branches added** — behavior now context-dependent
- **Performance optimization applied** (indexes added, query rewritten) — stories about performance may be satisfied differently

Action needed: **Review story acceptance criteria; may need AC updates but implementation can likely proceed with minor adjustments.**

#### **MINOR** — Cosmetic or additive changes
Changes unlikely to affect modernization value:
- **Logging statements added/modified** — no impact on business logic
- **Comments updated** — documentation improved
- **Code formatting/whitespace** — no functional change
- **New metrics/monitoring added** — observability improved
- **Unused code removed** — cleanup
- **Configuration comment clarified** — easier to understand existing behavior

Action needed: **Acknowledge but likely no story changes needed.**

#### **NEW SCOPE** — New files/components in the same module
- **New service class in same package** — may expand scope
- **New Kafka consumer or producer** — new integration point
- **New stored procedure in same package** — new business logic
- **New JSP or controller** — new entry point
- **New DAO layer** — new data access pattern

Action needed: **Evaluate whether modernization scope should expand; may spawn new stories.**

### 6. Build Impact Assessment Per Story

For each story in jira-stories.md:
- Identify all files it depends on (from the file→story mapping)
- Determine if any of those files have changed
- Calculate the **highest impact level** affecting that story (CRITICAL > SIGNIFICANT > MINOR)
- Summarize what changed and what action is needed

## Output Template

Create `drift-report.md` in the slug directory with this structure:

```markdown
# Drift Report: {slug}

**Generated**: {today's date} {current time}
**Report prepared for**: {slug} modernization initiative

---

## Summary

- **Baseline Commit**: `{hash}` ({branch}) on {timestamp}
- **Current Commit**: `{current-hash}` on {current-timestamp}
- **Commits Since Baseline**: {count}
- **Files Analyzed**: {count}
- **Files Changed**: {count}
- **Files Deleted**: {count}
- **New Files Detected**: {count}
- **Analysis Duration**: {seconds}

---

## Overall Drift Status

### 🟢 CLEAN
No changes detected. All stories remain valid and ready for implementation.

### 🟡 MINOR DRIFT
Only cosmetic or additive changes detected. Stories are valid; acknowledge findings and proceed.

### 🟠 SIGNIFICANT DRIFT
Business logic changes detected that may affect story acceptance criteria. Review recommendations before implementation.

### 🔴 CRITICAL DRIFT
File deletions or breaking changes detected. Stories must be updated before implementation can begin.

**Status**: {one of the above}

---

## Changed Files Detail

### 🔴 CRITICAL Changes

#### {filepath}

- **Status**: {DELETED | MODIFIED}
- **What Changed**: {2-3 sentence summary of the change}
  - {if deleted}: File removed entirely; any stories depending on classes/procedures in this file are at risk
  - {if modified}: Key method signatures changed; validation logic modified; schema altered; etc.

- **Git History**:
  - {commit-hash}: {commit message}
  - {commit-hash}: {commit message}
  - (include last 3-5 commits that touched this file)

- **Signature Changes**:
  - **Removed**: {method/procedure signature or table column}
  - **Modified**: {signature and what changed}
  - **Added**: {new signature or column}

- **Impacted Stories**:
  - {Story ID}: {brief reason why}
  - {Story ID}: {brief reason why}

- **Recommended Action**: {specific, actionable guidance}
  - Example: "Story ABC-123's AC#2 calls `OrderService.validateOrder(order)` with single parameter. Method signature now requires `(order, context)`. Update AC or modify story scope."
  - Example: "File OrderDAO.java deleted entirely. Story ABC-456 depends on `OrderDAO.findByStatus()` which no longer exists. Either restore file or rewrite story to use new DAO."

---

### 🟠 SIGNIFICANT Changes

#### {filepath}

- **Status**: MODIFIED
- **What Changed**: {description of the change}
- **Git History**:
  - {commit-hash}: {message}
  - ...
- **Signature Changes**:
  - {added/removed/modified signature}
- **Impacted Stories**:
  - {Story ID}: {brief reason}
- **Recommended Action**: {specific guidance}
  - Example: "Story ABC-789 tests 'Order can only be created if customer credit limit > order amount'. OrderService validation logic changed to also check 'order date is within business hours'. Update AC or add new requirement."

---

### 🟡 MINOR Changes

#### {filepath}

- **What Changed**: {brief description}
- **Impact**: {minimal / none}
- **Recommended Action**: No action needed; story remains valid.

(list all minor changes concisely)

---

### 🆕 New Scope Detected

| New File | Package/Location | Probable Role | Estimated Complexity | Recommendation |
|----------|------------------|---------------|----------------------|-----------------|
| {path} | {package} | {e.g., "Service class", "Kafka consumer"} | {Low/Medium/High} | {Expand scope / Investigate / Ignore} |
| ... | ... | ... | ... | ... |

---

## Story Impact Summary

| Story ID | Current Status | Files Affected | Highest Impact | Action Required | Risk Level |
|----------|---|---|---|---|---|
| {ID} | 🔴 At Risk | {count} files | CRITICAL | Must revise before implementation | 🔴 HIGH |
| {ID} | 🟡 Review | {count} files | SIGNIFICANT | Review ACs; may need minor updates | 🟡 MEDIUM |
| {ID} | 🟢 Clean | — | — | No action; ready to implement | 🟢 LOW |
| ... | ... | ... | ... | ... | ... |

**Stories Requiring Action**: {count}
**Stories Ready to Implement**: {count}

---

## Recommended Next Steps

1. {Prioritized action #1}
   - Who: {responsible team}
   - Timeline: {when}
   - Effort: {estimate}

2. {Prioritized action #2}
   - ...

3. {Prioritized action #3}
   - ...

### Examples of Recommended Actions:

- "Re-run `/analyze-entry` and `/gen-jira` for this slug to regenerate all stories from current state"
- "Only Story ABC-123 is affected. Re-run `/gen-jira` scoped to that story alone; others remain valid"
- "No CRITICAL issues. Proceed with implementation; schedule story review session for SIGNIFICANT items"
- "Investigate new files in {package}. Determine if modernization scope should expand"
- "Restore file {filepath} or confirm that it is dead code and can be excluded from modernization"

---

## Appendix: Change Detail

### Full Diff Snippets (Critical Changes Only)

#### {filepath}

\`\`\`diff
{git diff output, first 30 lines}
...
\`\`\`

---

## Sign-Off

- **Generated by**: /check-drift skill
- **Baseline**: `{slug}` fingerprint
- **Current State**: {branch} @ {current-hash}
- **Review Status**: ⬜ Not reviewed | 🟡 Pending review | ✅ Approved
```

## Guidelines

1. **Be Specific About Impact** — Don't just say "file changed". Explain what changed and why it matters for the story.

2. **Classify Rigorously** — use the four levels (CRITICAL/SIGNIFICANT/MINOR/NEW SCOPE) consistently. Err on the side of higher impact if uncertain.

3. **Link Changes to Stories** — always show which stories are affected and give developers the information they need to update them.

4. **Include Commit Messages** — the `git log` output explains *why* code changed; include it so stakeholders understand the context.

5. **Flag New Scope Clearly** — new files in the same package/directory are easy to miss; surface them explicitly for scope evaluation.

6. **Provide Actionable Guidance** — don't just say "story needs review". Say "Story ABC-123's AC#2 assumes method X(a, b); method now requires X(a, b, c). Either add context parameter to AC or update implementation plan."

7. **Distinguish Deletions from Modifications** — deleted files are always higher risk than modifications.

8. **Cross-Check Against Story Mappings** — the jira-stories.md file→story mapping is your source of truth for which stories are affected.

## Output Checklist

- [ ] Baseline commit is valid and reachable
- [ ] All fingerprinted files are analyzed
- [ ] Deleted files are marked CRITICAL
- [ ] Changed files are classified by impact level
- [ ] Signature changes are documented (removed, modified, added)
- [ ] Git commit history is included for each changed file
- [ ] New files in same scope are detected and listed
- [ ] Each changed file is cross-referenced to impacted stories
- [ ] Story Impact Summary table is complete (all stories listed with status)
- [ ] Recommended next steps are specific and prioritized
- [ ] Report identifies which stories need revision vs. which are ready to implement
- [ ] drift-report.md is written to `modernization-output/{slug}/`

## Error Handling

- **Baseline commit unreachable**: Report error and halt. Cannot proceed without valid baseline.
- **fingerprint.json missing or malformed**: Report error. Ask user to run `/analyze-entry` first.
- **jira-stories.md missing**: Drift is still detected and reported, but story impact mapping is unavailable. Note this limitation.
- **Git repository not found**: Report error. Drift detector requires git access to compute diffs and history.
- **File path issues** (Windows vs. Unix): Normalize paths using forward slashes; handle both separators gracefully.

