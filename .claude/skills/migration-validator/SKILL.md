---
name: validate-migration
description: Comprehensive validation of database migration from Oracle to Cosmos DB. Verifies schema completeness, data integrity, access patterns, change feed processors, and edge cases. Trigger on: validate migration, verify migration, migration check, data validation, migration completeness, schema completeness, post-migration, reconciliation. Produces validation-report.md with summary tables, verification matrices, and actionable findings.
---

# Migration Validator Skill (validate-migration)

## Overview

After a database migration from Oracle to Cosmos DB has been executed (or as a pre-flight dry-run check), this skill performs comprehensive validation to ensure the Cosmos DB data is complete, correct, and consistent with the Oracle source. It catches mapping gaps, data loss, constraint violations, and edge case handling issues before users encounter them in production.

Migration failures typically stem from:
1. **Schema gaps** — Oracle columns with no JSON equivalent
2. **Data loss** — Rows not migrated, counts don't match, embedded children missing
3. **Access pattern failures** — Cosmos DB queries returning wrong results or exceeding RU estimates
4. **Change feed gaps** — Event processors not deployed or not firing correctly
5. **Edge case bugs** — NULL/empty string confusion, numeric precision loss, timezone corruption

This skill catches all five by systematically verifying each aspect of the migration against ground truth (Oracle schema and data).

## Trigger

```
/validate-migration {db-slug}
```

**Parameters:**
- `{db-slug}` — directory name under `modernization-output/` containing schema-analysis.md, oracle-inventory.json, document-model.md, cosmos-config.json, and migration/ subdirectory

**When to Run:**
- Immediately after executing the migration script to verify correctness
- As a dry-run check before go-live
- When data reconciliation issues are reported
- To measure change feed processor success rate
- To validate edge case handling

## Inputs

The skill reads from:

1. **`modernization-output/{db-slug}/schema-analysis.md`** — Oracle table/column inventory with constraints and data types (from user-provided DDL)
2. **`modernization-output/{db-slug}/oracle-inventory.json`** — Structured Oracle objects:
   - `tables`: full DDL catalog (columns, constraints)
   - `procedures`, `packages`, `functions`: PLSQL bodies
   - `dependency_gaps`: tables/procedures referenced but not yet provided
   - `out_of_scope_refs`: triggers, sequences, views, jobs, synonyms, types (only flagged, not provided as full objects)
3. **`modernization-output/{db-slug}/document-model.md`** — Cosmos DB document structure with JSON paths, embedding decisions, and denormalization rules (including trigger → change feed mappings)
4. **`modernization-output/{db-slug}/cosmos-config.json`** — Cosmos DB container configuration (partition keys, TTL, indexing)
5. **`modernization-output/{db-slug}/migration/`** subdirectory:
   - `transform-scripts/` — Python transformation code showing type conversions
   - `export-queries/` — Oracle SELECT statements for each table
   - `validation/` — Pre/post-migration checks and reconciliation scripts
   - `infrastructure/` — Bicep/Terraform templates and Azure Function code
   - `rollback-plan.md` — Cutover and rollback strategy
6. **Oracle database** — Access to source database to query current state
7. **Cosmos DB** — Access to target database to verify migration results

## Verification Logic — Five-Check Approach

### Pre-Flight: Out-of-Scope References & Dependency Gaps

Before running the five checks, scan oracle-inventory.json and report on:

1. **Dependency Gaps** — tables/procedures referenced but not yet provided
   - Flag with severity: MEDIUM
   - Example: "⚠ CUSTOMERS table referenced by FK in ORDERS but DDL not provided"
   - Note: Migration can proceed for provided objects; gaps don't block validation

2. **Out-of-Scope References** — triggers, sequences, views, jobs, synonyms, types
   - For **sequences**: Confirm ID generation strategy used (GUID/ULID) and document in validation report
   - For **triggers**: Confirm change feed processors implement the trigger logic; validate in Check 4 below
   - For **views**: Note which views have been mapped to denormalized documents or query patterns
   - For **jobs, synonyms, types**: Flag for awareness; no validation needed unless explicitly mapped

### Check 1: Schema Completeness

For every Oracle table, column, and constraint:

1. **Table Mapping**
   - Read oracle-inventory.json for complete table list
   - For each table, verify it appears in document-model.md
   - Confirm mapping decision (dedicated container, embedded document, or denormalized)
   - Flag any Oracle table with no mapping

2. **Column Mapping**
   - For each column in the Oracle table, verify it appears in the document model
   - Check the type conversion (NUMBER → number, VARCHAR2 → string, DATE → ISO timestamp, CLOB → string or reference, BLOB → reference)
   - Verify the JSON property path is correct and follows naming conventions
   - Flag any Oracle column with no mapped JSON property
   - Note columns with intentional exclusions (document for removal rationale)

3. **Constraint Verification**
   - List all Oracle constraints (PRIMARY KEY, UNIQUE, NOT NULL, CHECK, FOREIGN KEY)
   - For each constraint, verify enforcement method:
     - **Cosmos DB enforced** — Unique index, partition key immutability, or TTL policy
     - **App-layer enforced** — Document in app code where validation occurs
     - **Removed intentionally** — Document business decision and approval
   - Flag any constraint without documented enforcement

### Check 2: Data Integrity (Post-Migration)

**Only applicable if Cosmos DB contains migrated data; if dry-run, skip this check.**

1. **Row Count Verification**
   - Query Oracle: `SELECT COUNT(*) FROM {table}` for each source table
   - Query Cosmos DB: Document count for equivalent container
   - Compare counts; flag any mismatch
   - Note: embedded children won't have separate counts; verify via embedded array count instead

2. **Sample Record Verification**
   - Randomly select N records from Oracle (N = 10-20 or 5% of data, whichever is smaller for large tables)
   - For each sample record:
     - Query Oracle to fetch full row
     - Query Cosmos DB to fetch corresponding document
     - Compare field-by-field:
       - **Data type** — is it the expected JSON type?
       - **Precision** — do numeric values match (no rounding errors)?
       - **Formatting** — dates/timestamps have correct format and timezone?
       - **NULL handling** — NULLs are missing properties or null values as per spec?
     - Flag any mismatches

3. **Embedded Document Verification**
   - For tables denormalized into parent documents (e.g., ORDER_ITEMS embedded in ORDERS):
     - Count children in Oracle: `SELECT COUNT(*) FROM order_items WHERE order_id = {id}`
     - Count embedded children in Cosmos DB: verify `order.items.length` for same order
     - Spot-check 5-10 parent documents to confirm embedded counts match

4. **Denormalized Field Consistency**
   - Identify any denormalized columns (e.g., customer_name repeated in order document for query performance)
   - For samples, verify denormalized copies match the source of truth
   - Flag any stale or mismatched denormalized data

### Check 3: Access Pattern Validation

For each access pattern documented in access-patterns.md:

1. **Query Execution**
   - Extract the Cosmos DB query (SQL or parameterized)
   - Execute it against live Cosmos DB
   - Verify it succeeds (no syntax errors, permissions granted, partition key correct)
   - Flag any query that fails

2. **Result Correctness**
   - For the query, construct the Oracle equivalent
   - Execute both queries and compare result sets
   - Verify same number of rows returned
   - Spot-check row content for correctness
   - Flag any result mismatch

3. **Performance Measurement**
   - Measure RU consumption for each Cosmos DB query
   - Compare against pre-migration estimate from architecture document
   - If actual > estimate by >20%, flag for optimization review
   - Document baseline for future performance tracking

### Check 4: Change Feed Processor Validation

For each Azure Function in migration/infrastructure/azure-functions/ (these replace Oracle triggers mapped in document-model.md):

1. **Deployment Verification**
   - Verify the Function App is deployed and running
   - Verify connection to the correct Cosmos DB account and container
   - Verify lease collection exists and is healthy
   - Flag any deployment or connectivity issues

2. **Trigger Verification**
   - **Important**: Oracle trigger bodies are not available (flagged as `out_of_scope_refs`). Change feed processors implement the business logic inferred from the document model, but must be confirmed with the application team.
   - Execute a test write to the monitored container (insert or update a test document)
   - Verify the change feed processor detects the change within 10 seconds
   - Verify the processor executes the intended side effect:
     - Does it call the correct downstream service?
     - Does it produce the expected output (audit log, calculated field, downstream message)?
   - **Confirm with team**: For each processor replacing an Oracle trigger, verify that the implemented logic matches the intended behavior. Flag any gaps: "Processor for {ORACLE_TRIGGER_NAME}: behavior not verified with team"

3. **Error Handling**
   - Verify the processor has dead-letter queue or retry logic for failures
   - Test failure scenario: inject a poison pill document or downstream service error
   - Verify processor recovers and doesn't block the lease

### Check 5: Edge Cases

1. **NULL Handling**
   - Oracle NULL vs JSON null vs missing property — verify consistency
   - Query Oracle: `SELECT col FROM table WHERE col IS NULL`
   - Query Cosmos DB: verify those documents have `col: null` or missing property (per spec)
   - Verify queries with `IS NULL` or filter `col = null` work correctly

2. **Empty String vs NULL**
   - Oracle: empty string '' is converted to NULL by some contexts
   - JSON: distinguish between `col: ""` and `col: null` and missing col
   - Verify the migration correctly distinguishes these cases
   - Query Oracle for empty strings: `SELECT col FROM table WHERE col = ''`
   - Verify Cosmos DB has appropriate representation and queries filter correctly

3. **Numeric Precision**
   - Oracle NUMBER(38,10) — very large or very precise decimal values
   - JSON number — loss of precision is possible
   - Sample verification: extract NUMBER(38,10) columns, verify no rounding or loss in Cosmos DB
   - Flag any precision loss

4. **Date/Timestamp/Timezone**
   - Oracle: DATE, TIMESTAMP, TIMESTAMP WITH TIME ZONE
   - JSON: ISO 8601 string with timezone info
   - Verify timezone offsets are preserved
   - Query Oracle for records with various timezones; verify Cosmos DB representation is correct
   - Verify queries filtering by date/time work correctly

5. **LOB Data (Large Objects)**
   - Oracle: CLOB (character large objects), BLOB (binary large objects)
   - Migration strategy: inline into JSON (if <1 MB) or store reference (if larger)
   - Verify:
     - Small CLOBs are inlined as strings
     - Large BLOBs are referenced (URL or storage reference), and references resolve
     - Inlined data has no corruption (encoding, special characters)
   - Flag any LOB data corruption

6. **Unicode & Multi-Byte Characters**
   - Test with non-ASCII data: accented characters, emoji, CJK characters
   - Query Oracle: extract records with Unicode data
   - Verify Cosmos DB preserves exact characters (no encoding loss, mojibake, etc.)
   - Flag any character corruption

## Output — `validation-report.md`

Create this file in the `modernization-output/{db-slug}/` directory.

### Structure

```markdown
# Migration Validation Report: {db-slug}

**Generated**: {ISO timestamp}
**Migration Initiative**: {db-slug}
**Oracle Source**: {connection string or description}
**Cosmos DB Target**: {connection string or description}
**Data Status**: {dry-run (no data yet) | post-migration (data present)}

---

## Executive Summary

- **Oracle Objects Mapped**: {count}/{total} ({%})
- **Column Mappings Verified**: {count}/{total} ({%})
- **Access Patterns Verified**: {count}/{total} ({%})
- **Change Feed Processors Verified**: {count}/{total} ({%})
- **Edge Cases Tested**: {count}/{total}
- **Critical Issues Found**: {count}
- **Warnings**: {count}

## Overall Status

**Color-Coded Status:**
- 🟢 **PASS** — All checks passed; migration is complete and correct; safe to go live
- 🟡 **WARNINGS** — Minor issues found (non-critical, can be resolved before production)
- 🔴 **FAILURES** — Critical issues detected; do not promote to production until resolved

**Current Status**: {🟢 PASS | 🟡 WARNINGS | 🔴 FAILURES}

---

## Pre-Flight: Out-of-Scope References

| Object Type | Count | How Handled | Status | Notes |
|---|---|---|---|---|
| Sequences | {count} | GUID/ULID generation in transform scripts | ✅ OK | Confirm ordering behavior with team if needed |
| Triggers | {count} | Change feed processors (see Check 4) | ⚠️ Verify | Trigger bodies not available — behavior must be confirmed with team |
| Views | {count} | Denormalized docs or query patterns | ⚠️ Review | Confirm views mapped to intended documents/queries |
| Jobs | {count} | Not migrated | ⚠️ Plan separately | No equivalent in Cosmos DB — plan alternative (Azure Functions, Logic Apps) |
| Synonyms | {count} | Not migrated | ⚠️ Plan separately | Refactor application code to use direct references |
| Types | {count} | Not migrated | ⚠️ Plan separately | Map to JSON document structures manually |

---

## Check 1: Schema Completeness Matrix

| Oracle Object Type | Count | Mapped | Complete | Status | Notes |
|---|---|---|---|---|---|
| Tables (in scope) | {count} | {count} ({%}) | {count} ({%}) | 🟢/🟡/🔴 | — |
| Columns (total) | {count} | {count} ({%}) | {count} ({%}) | 🟢/🟡/🔴 | — |
| Constraints (total) | {count} | Documented | {count} ({%}) enforced | 🟢/🟡/🔴 | List unenforced below |
| Dependency Gaps | {count} | — | — | ⚠️ | Warn if any remain; continue validation for provided objects |

### Missing/Incomplete Mappings

| Oracle Object | Expected JSON Path | Status | Recommendation |
|---|---|---|---|
| ORDERS.ORDER_DATE | orders.createdAt | ❌ Missing | Add to document model; update transform script |
| ORDERS.CURRENCY | orders.currency | ⚠️ Incomplete | Only mapped for USD orders; verify multi-currency handling |
| ORDER_ITEMS (constraint UNIQUE) | — | ❌ Unenforced | Implement app-layer validation in OrderService |

### Constraint Enforcement

| Constraint | Oracle Implementation | Cosmos DB / App Implementation | Status |
|---|---|---|---|
| ORDERS.PK | PRIMARY KEY (order_id) | Cosmos DB partition key + unique index on order_id | ✅ Enforced |
| ORDERS.TOTAL > 0 | CHECK constraint | App-layer validation in OrderService.validate() | ⚠️ Verify |
| ORDER_ITEMS.FK | FOREIGN KEY to ORDERS | No enforcement; relational integrity in app | ⚠️ Verify |

---

## Check 2: Data Integrity

**Data Status**: {dry-run | post-migration}

### Row Count Verification

| Oracle Table | Oracle Row Count | Cosmos DB Document Count | Match | Status |
|---|---|---|---|---|
| ORDERS | 5,000,000 | 5,000,000 | ✅ Yes | — |
| ORDER_ITEMS | 15,000,000 | (embedded) | ✅ Yes (avg 3 items/order) | — |
| CUSTOMERS | 2,500,000 | 2,500,000 | ✅ Yes | — |
| PRODUCTS | 100,000 | 100,000 | ✅ Yes | — |

**Issues**: {None found | List any count mismatches}

### Sample Record Verification

**Samples Tested**: N random records; spot-checks on fields by type

| Field | Type | Oracle Sample | Cosmos DB Sample | Match | Notes |
|---|---|---|---|---|---|
| order_id | NUMBER(10) | 1000001 | 1000001 | ✅ | — |
| customer_id | NUMBER(10) | 5000042 | 5000042 | ✅ | — |
| order_date | DATE | 2024-03-15 | 2024-03-15T00:00:00Z | ✅ | Date to ISO timestamp conversion correct |
| total | NUMBER(10,2) | 1234.56 | 1234.56 | ✅ | Decimal precision maintained |
| customer_name | VARCHAR2(100) | John O'Brien | John O'Brien | ✅ | Apostrophe preserved |
| description | CLOB | {truncated to 100 chars} | {same} | ✅ | CLOB inlined as string |

**Mismatches Found**: {None | List any field-level discrepancies}

### Embedded Document Verification

| Parent Table | Child Table | Sample Parent ID | Oracle Child Count | Cosmos DB Embedded Count | Match | Status |
|---|---|---|---|---|---|
| ORDERS | ORDER_ITEMS | 1000001 | 5 | 5 | ✅ Yes | — |
| ORDERS | ORDER_ITEMS | 1000042 | 3 | 3 | ✅ Yes | — |
| CUSTOMERS | CUSTOMER_ADDRESSES | 5000001 | 2 | 2 | ✅ Yes | — |

**Issues**: {None found | List any embedded count mismatches}

### Denormalized Data Consistency

| Document | Denormalized Field | Source of Truth | Sample Values Match | Status |
|---|---|---|---|---|
| orders | customer_name | customers container | ✅ Yes (5 samples checked) | — |
| orders | order_status_label | order_statuses lookup | ✅ Yes (5 samples checked) | — |

---

## Check 3: Access Pattern Validation

### Query Execution & Correctness

| Pattern ID | Query | Cosmos DB Status | Result Match | RU Consumption | Estimate | Variance | Status |
|---|---|---|---|---|---|---|
| AP-001 | Get order by ID | ✅ Success | ✅ Match | 2.5 RU | 2 RU | +25% | ⚠️ Review |
| AP-002 | Get orders by customer (last 30) | ✅ Success | ✅ Match | 8.3 RU | 10 RU | −17% | ✅ OK |
| AP-003 | Get orders in status=PENDING | ✅ Success | ✅ Match | 45 RU | 40 RU | +12% | ✅ OK |
| AP-004 | Get top 100 orders by date | ✅ Success | ✅ Match | 12 RU | 15 RU | −20% | ✅ OK |

**Query Failures**: {None | List any failed queries with error details}

**Result Mismatches**: {None | List any query result discrepancies}

**Performance Notes**:
- AP-001 is slightly over estimate; consider cross-partition query optimization
- All other patterns within acceptable variance; no immediate optimization needed

---

## Check 4: Change Feed Processor Validation

| Function | Container | Deployment Status | Connectivity | Test Write | Processor Fired | Correct Output | Status |
|---|---|---|---|---|---|---|
| fn-order-audit | orders | ✅ Deployed | ✅ Connected | ✅ Success | ✅ Yes (3.2s) | ✅ Yes | 🟢 PASS |
| fn-calc-total | orders | ✅ Deployed | ✅ Connected | ✅ Success | ✅ Yes (2.8s) | ✅ Yes | 🟢 PASS |
| fn-notify-warehouse | orders | ✅ Deployed | ✅ Connected | ✅ Success | ✅ Yes (4.5s) | ✅ Yes | 🟢 PASS |

**Deployment Issues**: {None found | List any deployment or connectivity failures}

**Processor Failures**: {None found | List any processors that don't fire or produce wrong output}

**Dead-Letter Handling**: All processors tested with poison pill; all handle gracefully and don't block lease.

---

## Check 5: Edge Cases

### NULL Handling

| Column | Oracle Nulls Found | Cosmos DB Representation | Query Correctness | Status |
|---|---|---|---|---|
| ORDERS.notes | Yes (1,234 rows) | col: null | ✅ Queries with `col = null` return correct results | ✅ OK |
| PRODUCTS.description | Yes (5,432 rows) | Missing property (per spec) | ✅ Queries with `NOT IS_DEFINED(col)` work | ✅ OK |
| CUSTOMERS.phone | Yes (234,543 rows) | null or missing (inconsistent) | ⚠️ Some queries miss results | 🟡 WARNING |

**Issues**: CUSTOMERS.phone has inconsistent null representation; recommend standardize to either null or missing.

### Empty String vs NULL

| Column | Oracle Empty Strings | Cosmos DB Representation | Queries Correct | Status |
|---|---|---|---|---|
| ORDERS.special_instructions | 892 rows | "" (empty string) | ✅ Correct | ✅ OK |
| PRODUCTS.alternate_code | 0 rows | — | — | ✅ OK |

**Issues**: None found.

### Numeric Precision

| Oracle Type | Column | Max Value | Cosmos DB Representation | Precision Lost | Status |
|---|---|---|---|---|
| NUMBER(38,10) | ORDERS.extended_price | 999999999999999999999999999.9999999999 | {actual value} | ❌ Yes (loss beyond 15 significant digits) | 🟡 WARNING |
| NUMBER(10,2) | ORDERS.total | {sample values} | {verified correct} | ❌ No | ✅ OK |

**Issues**: NUMBER(38,10) fields lose precision due to JSON number limitations. Recommend storing as strings if precision is critical.

### Date/Timestamp/Timezone

| Oracle Type | Sample Value | Cosmos DB ISO String | Timezone Preserved | Query Filter Works | Status |
|---|---|---|---|---|
| DATE | 2024-03-15 | 2024-03-15T00:00:00Z | ✅ Yes (UTC assumed) | ✅ Yes | ✅ OK |
| TIMESTAMP | 2024-03-15 14:30:45 | 2024-03-15T14:30:45Z | ✅ Yes (UTC assumed) | ✅ Yes | ✅ OK |
| TIMESTAMP WITH TZ | 2024-03-15 14:30:45-05:00 | 2024-03-15T14:30:45-05:00 | ✅ Yes (offset preserved) | ✅ Yes | ✅ OK |

**Issues**: None found.

### LOB Data (Large Objects)

| Oracle Column | Type | Sample Size | Migration Strategy | Cosmos DB Representation | Corruption Check | Status |
|---|---|---|---|---|---|
| ORDERS.notes | CLOB | <1 MB | Inline | String in JSON | ✅ No corruption | ✅ OK |
| PRODUCTS.image | BLOB | 2-50 MB | Reference | Azure Storage URL | ✅ Links valid, no corruption | ✅ OK |
| CUSTOMERS.bio | CLOB | <500 KB | Inline | String in JSON | ✅ No corruption | ✅ OK |

**Issues**: None found.

### Unicode & Multi-Byte Characters

| Sample Data | Oracle Value | Cosmos DB Value | Match | Status |
|---|---|---|---|---|
| Accented names | François | François | ✅ Yes | ✅ OK |
| Emoji | 🚀 Rocket | 🚀 Rocket | ✅ Yes | ✅ OK |
| CJK (Chinese) | 中文字符 | 中文字符 | ✅ Yes | ✅ OK |

**Issues**: None found.

---

## Recommendations & Action Items

### Out-of-Scope Objects — Plan Separately (Before Production)

For each out-of-scope reference found in oracle-inventory.json:

1. **Sequences** → Confirm GUID/ULID ID strategy is acceptable. If app depends on sequence ordering, implement alternative (sort by timestamp, add sequence-like counter field).

2. **Triggers** → For each trigger mapped to a change feed processor, verify the processor logic with the application team. Document any gaps: "Processor for {TRIGGER_NAME} — logic not confirmed."

3. **Views** → For each view, confirm its mapping: either a denormalized document, a query pattern, or a separate implementation. Flag any views without a documented mapping strategy.

4. **Jobs** → Plan alternative: Azure Functions, Logic Apps, Azure Data Factory, or application-layer scheduling.

5. **Synonyms** → Refactor application code to use direct references. Document in code migration plan.

6. **Types** → Map Oracle user-defined types to JSON document structures. Document in schema mapping.

### Critical (Block Go-Live)

{List any critical issues here, if any; otherwise state "None found."}

Example:
1. **CUSTOMERS.phone inconsistent NULL representation**
   - Impact: Query results may be incomplete
   - Fix: Standardize to either `null` or missing property; re-run migration for this column
   - Effort: 1-2 story points

### High Priority (Address Before Production)

{List any high-priority warnings, if any; otherwise state "None found."}

Example:
1. **NUMBER(38,10) precision loss**
   - Impact: Financial calculations may be incorrect for very large values
   - Decision: Store as strings in Cosmos DB if precision > 15 significant digits; update queries
   - Effort: 2-3 story points

2. **AP-001 RU consumption 25% over estimate**
   - Impact: Cost may exceed budget; performance degradation under load
   - Optimization: Review partition key strategy; consider cross-partition query rewrite
   - Effort: 2-3 story points

### Medium Priority (Address Within 30 Days)

{List any medium-priority findings; otherwise state "None found."}

---

## Validation Artifacts

- **Duration**: {start time} to {end time}
- **Verification Scope**: Schema, data, queries, processors, edge cases
- **Test Data Volume**: {N samples for data verification, {N} records for row count check, {N} access patterns tested}
- **Processor Test Coverage**: {count}/{count} Functions tested end-to-end

---

## Sign-Off

Before go-live, obtain approval from:

- [ ] **Data Architect** — confirms schema mapping is complete and correct
- [ ] **DBA/Platform Lead** — confirms data migration was successful and Cosmos DB is healthy
- [ ] **QA Lead** — confirms test coverage for all verified patterns
- [ ] **Application Owner** — confirms business requirements met and change feeds operating
- [ ] **Security/Compliance** — confirms data protection, encryption, and audit trails preserved

---

## Appendix: Validation Scripts Used

*Document all queries, scripts, and tools executed during validation to aid reproducibility.*

| Check | Script/Query | Purpose | Location |
|---|---|---|---|
| Schema | oracle-inventory.json parser | Extract table and column list | modernization-output/{db-slug}/oracle-inventory.json |
| Row Count | SELECT COUNT(*) FROM {table} | Verify record counts | Oracle source |
| Sample Data | Cosmos DB document retrieval | Fetch and compare sample records | Cosmos DB API |
| Access Pattern | {pattern query} | Test query execution and RU consumption | modernization-output/{db-slug}/document-model.md |
| Change Feed | Azure Function test invocation | Verify processor deployment and firing | migration/infrastructure/azure-functions/ |

```

---

## Guidelines for the Skill

1. **Be Exhaustive** — Verify every table, column, and constraint. Spot-check data across row count ranges. Don't assume correctness.

2. **Distinguish Schema from Data** — Schema completeness is verified pre-migration or with no data. Data integrity is only verified post-migration when data is present.

3. **Catch Edge Cases Early** — NULL, empty string, precision, timezone, Unicode. These cause subtle bugs in production. Test them explicitly.

4. **Measure Against Estimates** — RU consumption, query performance. Document variance so the team can optimize.

5. **Trace Every Issue** — For each finding, document:
   - What was checked (Oracle object, Cosmos DB equivalent)
   - What was found (mismatch, failure, warning)
   - Where it happened (table, column, query, function)
   - Why it matters (data loss risk, query failure risk, performance risk)
   - What to do (fix, remove, clarify, optimize)

6. **Change Feed Testing is Critical** — Processors must fire reliably. Test with actual writes; confirm side effects happen. One missing processor silently breaks downstream systems.

7. **Color Coding** — Use 🟢 🟡 🔴 consistently. Quick visual scan shows status at a glance.

8. **Be Actionable** — Don't just list failures. Recommend specific remediation steps with effort estimates and risk context.

## Output Checklist

- [ ] `validation-report.md` created in `modernization-output/{db-slug}/`
- [ ] Executive Summary includes all five checks
- [ ] Overall Status is clearly marked (🟢 🟡 🔴)
- [ ] Check 1 (Schema Completeness) verifies every table, column, and constraint
- [ ] Check 2 (Data Integrity) includes row counts, sample records, embedded documents, and denormalization checks (if post-migration)
- [ ] Check 3 (Access Patterns) tests query execution, result correctness, and RU consumption
- [ ] Check 4 (Change Feeds) verifies deployment, connectivity, and processor firing
- [ ] Check 5 (Edge Cases) covers NULL, empty string, precision, timezone, LOB, and Unicode
- [ ] All findings include Oracle source location and Cosmos DB equivalent
- [ ] Recommendations are prioritized (Critical, High, Medium) and actionable
- [ ] Sign-off section identifies approvers required before go-live
- [ ] Appendix documents all validation scripts and queries executed
