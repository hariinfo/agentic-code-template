---
name: scan-orphans
description: Scan entire codebase for orphaned, unclaimed, or dead code missed by entry-point analysis. Identifies active orphans (REST endpoints, Kafka consumers, scheduled jobs, stored procedures with no entry point coverage), classifies code as Active, Dead, or Uncertain, extracts functionality details, assesses risk, and recommends modernization path.
---

# Orphan Scanner (scan-orphans)

## Overview

After individual entry-point analyses have been completed, this skill performs a **codebase-wide sweep** to identify files and components that fall outside the entry-point tracing model. These "orphans" are code that may be deployed and in use — by external systems, database triggers, application server configurations, or manual processes — but invisible to entry-point-driven analysis.

The orphan scanner is the safety net that ensures complete modernization coverage.

## Inputs

You will work with:
- All existing `modernization-output/{slug}/fingerprint.json` files (to identify already-claimed code)
- The complete legacy codebase on disk
- Git history (commit timestamps) for activity assessment
- Configuration files: `web.xml`, Spring XML configs, Kafka consumer groups

## Process

### 1. Build the "Claimed Files" Set

Load each `fingerprint.json` file from all existing modernization-output slugs.

For each fingerprint.json, extract the complete file manifest:
```json
{
  "entry_point": "...",
  "files": {
    "java": [...all Java files discovered...],
    "jsp": [...all JSP files...],
    "sql": [...all SQL/PLSQL files...],
    "xml": [...all config XML files...],
    "properties": [...all property files...],
    "other": [...]
  }
}
```

Create a unified set of all claimed file paths. This is your "known good" set.

### 2. Scan Entire Codebase for Unclaimed Files

Recursively scan the legacy codebase for all source files:
- **Java** (`.java`) — all classes, interfaces, enums, annotations
- **JSP** (`.jsp`, `.jspx`) — all web pages and includes
- **SQL** (`.sql`, `.plsql`) — all stored procedures, functions, views, triggers
- **Configuration** (`web.xml`, `pom.xml`, `*.xml` in Spring config paths) — servlet definitions, filters, listeners
- **Properties** (`*.properties`) — configuration files
- **Other** — shell scripts, batch files, Kotlin, Groovy, etc.

For each file found:
- If it's in the claimed set → skip it
- If it's unclaimed → proceed to step 3

### 3. Classify Each Unclaimed File

For every unclaimed file, determine if it's **active orphan**, **dead code**, or **uncertain** by checking these indicators:

#### Indicators of Active Orphaned Code (Likely Still In Use)

1. **REST/Web Endpoints**:
   - Contains `@RestController` or `@Controller` annotation
   - Contains `@Path`, `@RequestMapping`, `@GetMapping`, `@PostMapping` etc.
   - Registered in `web.xml` as `<servlet>` or `<servlet-mapping>`
   - Referenced in Spring XML config files (e.g., `<bean class="...">`)

2. **Message Consumers**:
   - Contains `@KafkaListener`, `@JmsListener`, `@RabbitListener` annotation
   - Registered as message-driven bean in Spring config
   - Consumer group exists in Kafka broker (check if messages are flowing)

3. **Scheduled Jobs**:
   - Contains `@Scheduled` or `@Quartz` annotation
   - Registered as scheduled task in Spring config

4. **Standalone Programs**:
   - Has `public static void main(String[] args)` method
   - Deployed as batch job, CLI utility, or daemon process

5. **Database Objects**:
   - Is a stored procedure/function in `/db/procedures/` directory
   - NOT called from any DAO in the claimed files
   - But may be called by triggers, other procedures, or external ETL tools

6. **Recent Activity**:
   - Has git commits within the last 12 months
   - File modification timestamp is recent (within last 6 months)
   - This suggests active maintenance

#### Indicators of Dead Code (Likely Safe to Ignore)

1. **No Activity**:
   - No git commits in 2+ years
   - File modification timestamp older than 2 years

2. **Explicit Deprecation**:
   - Contains `@Deprecated` annotation
   - Contains comment "// DEPRECATED", "// DO NOT USE", "// LEGACY"

3. **Location-Based Signals**:
   - File path contains `_deprecated/`, `_old/`, `_archive/`, `legacy/` directory
   - File name contains `.deprecated`, `.old`, `.bak`, `.unused` suffix

4. **No References**:
   - Searched entire codebase (including unclaimed files) with grep
   - No references from any Java `import` statement, JSP `<%@ include %>`, or SQL stored procedure call
   - No references in configuration files or property files

#### Indicators of Uncertain Code (Needs Manual Review)

- Mix of active and dead signals (e.g., recent commit but no references, no `@Deprecated` but in `_old/` folder)
- Possible external integration (may be called by scripts, external systems, or ad-hoc queries)
- Configuration suggests it's used but code analysis can't confirm

### 4. For Each Active Orphan: Extract Full Details

Apply the same exhaustive extraction as analyze-entry step 12:

1. **Entry Point Identification**
   - Type: REST controller? Kafka consumer? Scheduled job? Stored procedure? Servlet filter?
   - Route/topic/schedule if applicable

2. **Behavioral Extraction**
   - Read the full method/procedure body
   - Extract all business logic, decision trees, state transitions
   - Identify inputs, outputs, side effects

3. **Data Dependencies**
   - Which tables does it read/write?
   - Which APIs does it call?
   - Which Kafka topics does it publish/consume?
   - Which configuration values does it depend on?

4. **Error Handling**
   - What exceptions are caught?
   - What's the retry/fallback strategy?
   - What's the error response or logging?

5. **Integration Points**
   - External APIs called
   - Kafka topics (in/out)
   - Database tables (read/write)
   - Cache operations

6. **Security & Permissions**
   - Required roles/permissions
   - Authentication mechanism
   - Data sensitivity

7. **Evidence of Use**
   - Git commit history and most recent commit date
   - References from other code (even unclaimed code)
   - Registered in web.xml or Spring config
   - Active message flow (for Kafka consumers)
   - Recently executed logs (if available)

### 5. Assess Risk for Each Active Orphan

For each active orphan, determine:

1. **Impact of Missing Modernization**
   - Does a user-facing feature break? (HIGH)
   - Does an integration point fail? (HIGH)
   - Does a background job stop running? (MEDIUM)
   - Does audit/compliance break? (HIGH)

2. **Complexity to Modernize**
   - Simple routing/forwarding? (LOW)
   - Business logic with 3+ decision trees? (MEDIUM)
   - Complex state machine? (HIGH)
   - Many external dependencies? (HIGH)

3. **Data Sensitivity**
   - PII/financial data? (HIGH)
   - Audit-critical data? (HIGH)
   - Public/internal-only data? (MEDIUM/LOW)

## Output Template

Create `orphan-report.md` in `modernization-output/` (root level, not in a slug directory):

```markdown
# Orphan Scan Report

**Generated**: {today's date and timestamp}

---

## Summary

- **Total codebase files scanned**: {count}
- **Files claimed by entry-point analyses**: {count} across {N} slugs
- **Unclaimed files**: {count}
- **Active orphans (likely still in use)**: {count}
- **Dead code (likely safe to ignore)**: {count}
- **Uncertain (needs manual review)**: {count}

**Key Finding**: {e.g., "5 critical REST endpoints missed in entry-point analysis" or "All major flows covered; orphans are primarily deprecated code"}

---

## 🔴 Active Orphans — Likely Still In Use

### {File: com/acme/payment/PaymentWebhookController.java}

**Type**: REST Endpoint (Webhook receiver)

**Location**: `/src/main/java/com/acme/payment/PaymentWebhookController.java`

**Evidence of Use**:
- Contains `@RestController` with `@PostMapping("/webhook/payment")`
- Git commits as recent as 2 weeks ago
- Referenced in `web.xml` via servlet mapping to this controller
- Kafka topic `payment.webhook.events` shows active message flow (5K messages/day)

**What It Does**:
- Receives webhooks from external payment processor (Stripe/Square)
- Validates HMAC signature of incoming request
- Parses payment status update (success, failure, refund)
- Updates ORDER_PAYMENT table with new status
- Publishes `PaymentStatusUpdated` Kafka event to downstream systems (order service, reporting)
- Returns 200 OK or 400 Bad Request
- Retries on database deadlock (up to 3 times with exponential backoff)

**Detailed Extraction**:

**Input Schema**:
```json
{
  "id": "evt_xxx",
  "type": "charge.succeeded|charge.failed|charge.refunded",
  "timestamp": "ISO 8601",
  "data": {
    "amount": number (cents),
    "currency": "USD",
    "order_id": string,
    "reference": string
  }
}
```

**Processing Logic**:
1. Verify HMAC signature using secret from `payment.webhook.secret` property
2. If signature invalid → return 400 Unauthorized
3. If type == "charge.succeeded" → set ORDER_PAYMENT.status = "CONFIRMED", call markOrderReady()
4. If type == "charge.failed" → set status = "FAILED", call notifyOrderFailed(), revert inventory hold
5. If type == "charge.refunded" → set status = "REFUNDED", reverse corresponding invoice
6. On database deadlock → exponential backoff (100ms, 500ms, 2.5s) then fail if still locked
7. Publish event to Kafka topic `payment.webhook.events` with full payload
8. Log event with timestamp, status, order_id to audit table

**Data Dependencies**:
- Reads: `WEBHOOK_KEYS` table for signature validation
- Writes: `ORDER_PAYMENT` table (status column)
- Reads: `ORDER` table (to fetch order details)
- Publishes: Kafka topic `payment.webhook.events`
- Config: `payment.webhook.secret`, `payment.max_retries`, `payment.retry_backoff_ms`

**Error Handling**:
- Missing order_id → 400 Bad Request, log warning
- Invalid signature → 400 Unauthorized, log security event
- Database locked → retry with backoff
- Kafka publish failure → log ERROR, leave webhook as unprocessed (external retry will happen)

**External Integrations**:
- **Inbound**: Payment processor webhooks (Stripe, Square, custom)
- **Outbound**: Kafka `payment.webhook.events` (consumed by order-service, reporting-service)
- **DB**: Direct update to ORDER_PAYMENT (shared table)

**Security**:
- HMAC signature validation required (prevents spoofing)
- No authentication beyond HMAC (webhooks are asynchronous push)
- Logs contain order_id but not amount/currency (PII consideration)

**Risk If Missed**:
- Payment confirmations won't reach the order system
- Customers' paid orders won't be fulfilled
- Revenue recognition breaks
- **CRITICAL** — this MUST be modernized

**Recommended Action**:
- Create a new `/analyze-entry` for this as its own entry point (payment-webhook-receiver)
- High priority for modernization (revenue-critical)
- Plan as Phase 1 if rebuilding payment flows

---

### {File: db/procedures/archive_old_orders.sql}

**Type**: Stored Procedure (Scheduled maintenance)

**Location**: `/db/procedures/archive_old_orders.sql`

**Evidence of Use**:
- Git commits 3 months ago (recent changes)
- Called from `OrderMaintenanceScheduler.java` via `CallableStatement`
- But `OrderMaintenanceScheduler.java` is NOT in any fingerprint (it's also an orphan!)
- No references from analyzed entry points

**What It Does**:
- Runs nightly (2 AM UTC) via Oracle job scheduler
- Moves orders older than 5 years to `ORDER_ARCHIVE` table
- Deletes detailed rows from ORDER_ITEMS, ORDER_AUDIT older than 5 years
- Rebuilds index on ORDERS table
- Logs archival counts to MAINTENANCE_LOG table

**Detailed Extraction**:

**Procedure Signature**:
```sql
PROCEDURE archive_old_orders(
  p_archive_date IN DATE DEFAULT TRUNC(SYSDATE - 5*365),
  p_dry_run IN BOOLEAN DEFAULT FALSE
) IS
```

**Logic**:
1. Calculate cutoff date (5 years ago)
2. If p_dry_run = TRUE:
   - Count affected rows, log counts, return without modifying
   - Used for operational safety checks
3. Else (real run):
   - Begin transaction with isolation level SERIALIZABLE
   - Insert from ORDERS WHERE created_date < cutoff into ORDER_ARCHIVE
   - Insert from ORDER_ITEMS WHERE order_id IN (...) into ORDER_ITEMS_ARCHIVE
   - Insert from ORDER_AUDIT WHERE order_id IN (...) into ORDER_AUDIT_ARCHIVE
   - Delete from ORDER_ITEMS, ORDER_AUDIT (rows now archived)
   - Delete from ORDERS (rows now in ORDER_ARCHIVE)
   - Rebuild index ORDERS_IDX_CREATED_DATE
   - Commit
4. On constraint violation (FK integrity) → raise error, log, rollback
5. Log summary (rows archived, elapsed time, status) to MAINTENANCE_LOG

**Data Impact**:
- Reads: ORDERS, ORDER_ITEMS, ORDER_AUDIT (rows older than 5 years)
- Writes: ORDER_ARCHIVE, ORDER_ITEMS_ARCHIVE, ORDER_AUDIT_ARCHIVE (appends)
- Deletes: ORDERS, ORDER_ITEMS, ORDER_AUDIT (rows older than 5 years)
- Affects indexes: ORDERS_IDX_CREATED_DATE

**External Dependencies**:
- Scheduled by Oracle job scheduler (not from app)
- Accessed by ad-hoc queries for archive verification

**Risk If Missed**:
- Orders table grows unbounded, query performance degrades
- Storage costs increase
- Data retention/compliance may require archival
- **MEDIUM-HIGH** — affects data warehouse health, not user-facing

**Recommended Action**:
- Modernize as part of data maintenance layer
- Implement as batch job (Spring Batch or similar) rather than stored procedure
- Keep as lower priority than user-facing endpoints, but before deployment cutover

---

(repeat for each active orphan)

---

## 🟡 Uncertain — Needs Manual Review

| File | Last Modified | Indicators | Possible Purpose |
|------|---------------|-----------|-----------------|
| `/src/main/java/com/acme/reporting/LegacyReportGenerator.java` | 2024-02-15 | Has `@Component` but no `@Scheduled`, no recent commits, no references found in code search | May be legacy reporting tool, replaced by BI system; verify with analytics team |
| `/db/procedures/legacy_data_sync.sql` | 2023-11-20 | Called by script, no app references, in `_legacy/` folder but not marked `@Deprecated` | May be external ETL dependency; check with data engineering |
| `/src/main/webapp/admin/debug.jsp` | 2024-01-10 | JSP with debugging endpoints, low commit frequency | May be internal diagnostic tool; verify if still needed in production |

---

## 🟢 Dead Code — Likely Safe to Ignore

| File | Last Modified | Reason Classified as Dead | Recommended Action |
|------|---------------|--------------------------|-------------------|
| `/src/main/java/com/acme/legacy/OldPaymentGateway.java` | 2020-07-22 | No commits since 2020, marked `@Deprecated`, zero references in codebase | Remove in next cleanup sprint |
| `/db/procedures/deprecated_report_v1.sql` | 2019-03-05 | In `_deprecated/` folder, no git activity, no references | Safe to delete |
| `/src/main/java/com/acme/experiment/ExperimentalCache.java` | 2021-11-18 | Commented out entire class, in `_old/` folder, no references | Remove with next refactor |

---

## Recommended Next Steps

### Immediate Actions (Before Full Modernization)

1. **Validate Active Orphans** (Within 1 week)
   - Schedule 30-min sync with team leads for each active orphan
   - Confirm each one is still in use (or mark for removal)
   - Document business owner/stakeholder for each

2. **Create New Entry Points** (Within 2 weeks)
   - For each confirmed active orphan, run `/analyze-entry <file-path>`
   - This brings them into the modernization pipeline
   - Example: `/analyze-entry /src/main/java/com/acme/payment/PaymentWebhookController.java`

3. **Resolve Uncertain Files** (Parallel with above)
   - For each uncertain file, investigate with team
   - Either move to active orphans (run /analyze-entry) or mark as dead code (schedule for removal)

### Sequencing for Modernization

**Phase 1 (Critical Path)**:
- {e.g., "PaymentWebhookController" — revenue-critical}
- {e.g., "Any orphaned auth/security endpoints"}

**Phase 2 (Supporting Systems)**:
- {e.g., "Scheduled maintenance jobs"}
- {e.g., "Kafka consumers not yet covered"}

**Phase 3 (Cleanup)**:
- After all active orphans are modernized, remove dead code
- Update build configs to exclude removed files

### Post-Scan Validation

After addressing all active orphans:

```bash
/scan-orphans  # Re-run to confirm full coverage
```

Expected result: Only uncertain/dead code remains. Zero active orphans.

---

## Key Insights

- **Total coverage before scan**: {N}% (from {count} entry-point analyses)
- **After modernizing active orphans**: {estimated M}% coverage
- **Dead code to be removed**: {count} files, {size in KB}
- **Recommended removal effort**: {e.g., "1-2 sprints for cleanup"}

---

## Appendix: Scan Methodology

This report was generated using automated codebase scanning with the following heuristics:

1. **Claimed files identification** — union of all `fingerprint.json` manifests
2. **Active indicators** — annotation detection (regex), git history analysis, configuration file cross-reference
3. **Dead code detection** — git timestamp analysis, deprecation markers, directory patterns, reference scanning
4. **Risk classification** — entry point type, data sensitivity, reference count, recent activity

Manual review is essential for uncertain classifications. Automated detection has limits (e.g., reflection-based calls, external system invocations not visible in source code).

```

## Guidelines

1. **Be Thorough on Active Orphans** — extract the same level of detail as analyze-entry produces, so teams can understand exactly what needs to be modernized

2. **Classify Conservatively** — when in doubt, mark as "Uncertain" rather than "Dead Code". It's safer to ask for clarification than to lose code.

3. **Show Evidence** — every classification (active/dead/uncertain) must cite specific evidence (git commits, annotations, references, configuration)

4. **Assess Real Risk** — don't just say "modernize this". Explain what breaks if you skip it (revenue? compliance? SLA?).

5. **Recommend Concrete Actions** — "Create new /analyze-entry for this file" is actionable. "Review with team" requires follow-up.

6. **Use Git History** — commit timestamps are your most reliable signal of active vs. dead code. Lean on them.

7. **Check Configuration** — web.xml, Spring XML, property files, and Kafka brokers often show what code is actually deployed, independent of source code references.

## Output Checklist

- [ ] orphan-report.md created at `modernization-output/orphan-report.md` (root level, not in slug)
- [ ] Summary counts are accurate (total files, claimed, unclaimed, active/dead/uncertain)
- [ ] Each active orphan has: type, evidence, extraction details, risk assessment, recommended action
- [ ] Each uncertain file includes: indicators table with possible purposes
- [ ] Each dead code entry includes: last modified date and reason for classification
- [ ] Recommended Next Steps section provides actionable guidance (specific commands, phase sequencing)
- [ ] For each active orphan, suggest running `/analyze-entry` to bring it into the pipeline
- [ ] Appendix explains the scan methodology and limitations
