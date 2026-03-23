---
name: design-documents
description: Design Cosmos DB NoSQL document model from Oracle schema analysis. Transforms relational schema (tables, constraints, PLSQL) into denormalized documents, containers, partition keys, and access patterns. Handles scoped inventory with dependency gaps and out-of-scope references (sequences, triggers, views). Produces document-model.md with JSON schemas, column mappings, constraint migration, trigger mapping, denormalization registry, and complete Cosmos DB configuration for migration from Oracle to Azure Cosmos DB.
---

# Document Model Designer (design-documents)

## Overview

This skill transforms an analyzed Oracle relational schema into a fully designed **Cosmos DB NoSQL document model**. Oracle → Cosmos DB is not a "port" — it's a complete redesign. Relational databases normalize data and join at query time. Cosmos DB denormalizes data and optimizes for access patterns at write time. This skill applies NoSQL design principles to create an optimal, implementable document model.

The design-documents skill is the intellectual core of the database modernization pipeline: it decides which tables embed together, how data denormalizes, where constraints move to the app layer, and how triggers become change feed handlers.

**Important**: This skill works with a **scoped inventory** from the analyze-schema skill. The inventory includes tables, constraints, and PLSQL code. References to sequences, triggers, views, and other objects are flagged in `dependency_gaps` and `out_of_scope_refs` arrays for design decisions without requiring full object definitions.

## Inputs

You will receive a **db-slug** — the directory name under `modernization-output/` containing:

- `schema-analysis.md` (required) — comprehensive Oracle schema analysis of provided DDL
- `oracle-inventory.json` (required) — machine-readable catalog of tables, constraints, and PLSQL procedures/functions/packages. May include `dependency_gaps` (referenced but not-yet-provided objects) and `out_of_scope_refs` (sequences, triggers, views, etc.)
- `relationship-map.md` (required) — entity relationships and suggested aggregates based on provided tables
- `source-extracts.md` (optional) — from code pipeline; reveals actual application query patterns
- `techspec.md` (optional) — from code pipeline; reveals transaction boundaries and caching

## Process

### Pre-Check — Validate Inventory Completeness

Before starting design work, verify the analysis is complete:

1. **Read `oracle-inventory.json`** and check for `dependency_gaps`:
   - If the `dependency_gaps` array is non-empty, this means tables or procedures referenced by the provided objects are not yet in the inventory
   - **Warn**: "⚠ Analysis is incomplete. {N} dependency gap(s) found. Some design decisions (esp. FK embedding strategies) may need revisiting once missing objects are analyzed. Recommend completing the analyze-schema step first."
   - Continue anyway (design can proceed with known objects), but flag assumptions

2. **Read `oracle-inventory.json`** and check for `out_of_scope_refs`:
   - These are references to sequences, triggers, views, jobs, synonyms, and other non-core objects
   - Note their locations for later design decisions (not an error — expected)
   - These will be handled in specific phases below

### Phase 1 — Identify Access Patterns

Before designing documents, catalog every way the data is accessed. This is the foundation of all design decisions. NoSQL design is access-pattern-driven.

**1. Extract Access Patterns from Artifacts**

- **From schema-analysis.md**: Every SELECT, INSERT, UPDATE, DELETE in stored procedures, views, and PLSQL packages
- **From oracle-inventory.json**: Query patterns embedded in procedure bodies and view definitions
- **From source-extracts.md** (if available): Actual queries the application executes — far more reliable than what "might" be used
- **From indexes**: Existing indexes reveal query patterns the DBA optimized for (column order matters)
- **From views**: Views reveal common read patterns that became materialized for performance

**2. Catalog Each Access Pattern**

For every pattern identified, document:
- **Operation type**: Read / Write / Update / Delete / Batch
- **Frequency**: Real-time user-facing / Batch / Reporting / Admin (rare)
- **Filter conditions**: WHERE clauses → primary partition key candidates
- **Sort requirements**: ORDER BY clauses → secondary sort key candidates
- **Tables involved**: Which JOINs — determines embedding decisions
- **Volume**: Rows returned/affected → throughput estimation
- **Latency requirement**: Interactive (< 100ms) vs. acceptable delay

**3. Identify Hot vs. Cold Access**

- Hot patterns: High frequency, low latency requirement → single partition preferred
- Cold patterns: Batch, reporting, rare admin → cross-partition acceptable
- This informs partition key choice and denormalization strategy

### Phase 2 — Design Document Model

Apply NoSQL design principles to transform the relational schema into an optimized document model.

**4. Determine Aggregate Boundaries**

Group tables that are always accessed together into single documents:

- **1:1 relationships** → almost always embed in parent
- **Parent-child 1:N with CASCADE delete** → strong candidate for embedding
- **1:N where child is accessed independently** → separate container with reference
- **1:N where N is very small (< 100) and always read with parent** → embed
- **1:N where N is large or frequently filtered** → separate container
- **N:M relationships** → embed on both sides (duplication) or use reference pattern

Review `relationship-map.md` for suggested aggregate boundaries — use as starting point.

**5. Choose Containers**

Cosmos DB containers are the logical equivalent of relational tables:

- Each major aggregate typically becomes one container
- Consider "single container with discriminator pattern" for entity hierarchies that share the same partition key
- Example: ORDERS + ORDER_HISTORY as separate document types in same orders container
- Target: minimize containers while maintaining clear, efficient access patterns

**6. Choose Partition Keys** — Most Critical Decision

The partition key determines throughput, latency, and cost:

- **Must exist on every document** in the container
- **Should distribute evenly** — avoid hot partitions (some partitions receiving 99% of traffic)
- **Should align with primary access filter** — queries within one partition are fast; cross-partition queries are expensive
- **Candidates to evaluate**: customer_id, tenant_id, region, order_date (year-month), category, facility_id
- **For each container**: evaluate 2-3 candidates with pros/cons documented

Pros/cons template:
- **Candidate A: /customerId**
  - ✓ Aligns with AP-002 (high-frequency query: get all orders for customer)
  - ✓ Good distribution (millions of distinct customers)
  - ✗ AP-003 (search by date range) becomes cross-partition expensive
  - Decision: ACCEPT

**7. Design Document Schemas**

For each container, define the JSON document structure:

- **Map Oracle columns → JSON properties** with type conversions
- **Embed child entities** as nested arrays/objects where appropriate
- **Add discriminator field** (`"type": "order"`) if mixing entity types in one container
- **Add partition key field** if it's synthetic (non-natural business key)
- **Preserve legacy IDs** (e.g., Oracle ORDER_ID) as `legacyOrderId` for migration traceability
- **Handle Oracle types**:
  - NUMBER(p,s) → number (or string if p > 15 for precision)
  - DATE/TIMESTAMP → ISO-8601 string
  - CLOB → string (or Azure Blob Storage reference if > 2MB)
  - BLOB → Azure Blob Storage reference with URL
  - RAW → base64 string
  - User-defined types → nested objects
  - INTERVAL → ISO 8601 duration string
  - BOOLEAN/Y-N flags → boolean

- **Add system metadata fields**: `_metadata: { createdAt, updatedAt, version }` for optimistic concurrency

**8. Handle Denormalization**

NoSQL requires intentional duplication. Document every duplication:

- **What gets duplicated?** (e.g., customer name embedded in order)
- **Why?** (query needs it without separate lookup)
- **Update strategy**:
  - Change feed propagation (eventual consistency)
  - Snapshot at write time (never updated)
  - Acceptable staleness (< 1 minute? < 1 hour?)
- **Flag high-change data** — data that changes frequently may be better as reference than embedded

Create **Denormalization Registry** table: duplicated field → source → embedded in → update strategy → staleness tolerance

**9. Handle Sequences → ID Generation**

Oracle sequences have no equivalent in Cosmos DB. Identify sequence references from `out_of_scope_refs` in the inventory:

- **Locate sequence references**: Check `out_of_scope_refs` array for entries of type "sequence" and which tables/columns use them
- **For each sequence reference**, document:
  - Which table(s) and column(s) use this sequence
  - Current Oracle sequence behavior (start value, increment, cycle)
  - **Default strategy**: GUID (v4) — simple, collision-free, no coordination
  - **Alternative**: ULID (sortable, time-based, preserves insertion order)
  - **Alternative**: Snowflake IDs (time-ordered, sortable, generated app-side)
  - **Recommendation**: Use GUID for new IDs, preserve legacy Oracle sequence IDs as `legacyId` during migration
- If a sequence is not referenced in provided objects but exists in oracle-inventory.json, note it as unused

**Sequences Table Example**:
| Oracle Sequence | Used By Table | New Strategy | Rationale | Legacy Migration |
|-----------------|---------------|-------------|-----------|-----------------|
| ORDER_SEQ | ORDERS.ORDER_ID | GUID v4 | No coordination needed, no hot spots | Preserve as `legacyOrderId` during migration |
| ITEM_SEQ | ORDER_ITEMS.ITEM_ID | ULID | Sortable by time, preserves insertion order | Not used in app — safe to replace |

**10. Handle Disappearing Constraints**

Relational constraints become app-layer responsibilities:

- **Foreign keys** → app-layer referential integrity validation
- **Unique constraints** → Cosmos DB unique key policy (within partition) OR app-layer check (cross-partition)
- **Check constraints** (e.g., `TOTAL > 0`) → app-layer validation in service layer
- **NOT NULL** → app-layer validation (Cosmos DB is schema-free)
- **Column defaults** → app-layer defaults before insert

Document every constraint: Oracle constraint → Cosmos DB handling → which service layer enforces it

**11. Handle Triggers**

Oracle triggers contain invisible business logic. Trigger references come from `out_of_scope_refs` in the inventory (trigger bodies are not provided unless explicitly included):

**Process**:

1. **Locate trigger references** in `out_of_scope_refs` — identify which triggers are referenced by which tables or PLSQL objects
2. **For each trigger reference**, even without the full trigger body, document:
   - **Trigger name** and source table/event (AFTER INSERT/UPDATE/DELETE)
   - **Inferred purpose** from context: audit, cascade update, validation, computed column
   - **Mapping to Cosmos DB**:
     - **Audit triggers** → Change feed + Azure Function writes to audit container
     - **Cascade update triggers** → Change feed + Azure Function for propagation
     - **Validation triggers** → App-layer validation before write
     - **Computed column triggers** → App-layer computation before write
     - **Enforcement triggers** (referential integrity, uniqueness) → App-layer validation
   - **Change feed processor name** (e.g., `orders-audit-processor`)
   - **Azure Function name** (e.g., `fn-order-audit`)
   - **Target action**: what the trigger does
   - **⚠ Flag**: If trigger body is unavailable → "Change feed handler needed — trigger body not available, logic TBD"

**Trigger Mapping Example**:
| Oracle Trigger | Event | Inferred Purpose | Cosmos DB Handler | Target Action | Body Available |
|---------------|-------|------------------|------------------|--------------|-----------------|
| TRG_ORDER_AUDIT | AFTER INSERT/UPDATE | Audit changes | orders-audit-processor → fn-order-audit | Write audit record to audit-events container | ✓ Yes |
| TRG_ORDER_VALIDATE | BEFORE INSERT | Validation | N/A | App-layer validation in OrderService.validate() | ✗ No — logic TBD |
| TRG_CUSTOMER_UPDATE | AFTER UPDATE | Cascade propagation | customers-feed-processor → fn-order-update-customer-name | Update customerName in orders container | ✗ No — inferred from denormalization needs |

**12. Design Indexing Policy**

Cosmos DB indexes all paths by default (inclusive). Customize for performance:

- **Default**: `"indexingMode": "consistent"` — all queries always return up-to-date results
- **For hot paths**: Include in index (needed for WHERE, ORDER BY, JOIN)
- **For cold paths**: Exclude from index (not queried, saves cost)
- **Composite indexes**: For ORDER BY on multiple fields (e.g., `/customerId` + `/orderDate`)
- **Range indexes**: For numeric/string comparisons
- **Spatial indexes**: If geo queries (point/polygon)

Provide complete `indexingPolicy` JSON for each container.

**13. Estimate Throughput (RU/s)**

Request Units (RUs) estimate cost and latency:

- **Read 1KB document** = ~1 RU
- **Write 1KB document** = ~5-10 RU
- **Cross-partition query** = 1 RU + cost of items returned
- **For each container**: estimate read/write volume from access patterns
- **Recommend**: Autoscale (flexible), Serverless (pay per request), or Provisioned (fixed)

Example:
- AP-002 (get orders for customer): 1 RU point read × 1000/sec = 1000 RU/s reads
- Writes (new orders): ~10 RU × 100/sec = 1000 RU/s writes
- **Recommendation**: Autoscale 2000-6000 RU/s

### Phase 3 — Validate Design

**14. Verify Every Access Pattern is Satisfied**

For each access pattern from Phase 1:
- Can it be served by the document model? (single partition / single query / multiple queries?)
- Does it require cross-partition query? (expensive — consider denormalization)
- Does it require multiple queries / JOINs? (no JOINs in Cosmos DB — may need further embedding)
- Document mitigation for expensive patterns

**15. Verify Every Provided Object is Accounted For**

Traceability is critical for migration. Work with the scoped inventory:
- Every **provided table** → mapped to container / document type (or explicitly excluded with reason)
- Every **provided column** → mapped to JSON property or excluded with reason
- Every **provided constraint** → mapped to Cosmos DB feature or app-layer enforcement
- Every **provided PLSQL procedure/function/package** → mapped to app-layer service or migration strategy
- Every **out-of-scope trigger reference** → mapped to change feed handler or app-layer logic with note on body availability
- Every **out-of-scope view reference** → mapped to denormalized document or query pattern (note: view body not available)
- Every **out-of-scope sequence reference** → mapped to ID strategy
- Every **dependency gap** (referenced but not-yet-provided) → flagged as requiring future analysis for final design decisions

## Output Templates

### Output 1 — `document-model.md`

```markdown
# Cosmos DB Document Model: {schema-name}

**Database**: {db-name}
**Generated**: {today's date}
**Source**: {db-slug path}

## Design Summary

| Metric | Value |
|--------|-------|
| Containers | {count} |
| Document Types | {count} |
| Embedded Relationships | {count} |
| Reference Relationships | {count} |
| Denormalized Fields | {count} |
| Constraints Moved to App Layer | {count} |
| Triggers → Change Feed | {count} |

## Container: {container-name}

### Purpose
{What this container stores — e.g., "Core order and order item data with denormalized customer and product info"}

### Partition Key: `/{partition-key-path}`

**Rationale**: {Why this key was chosen based on access patterns}

**Alternatives Considered**:
- `/orderId`: Distributes evenly but breaks AP-002 (get orders for customer), would be cross-partition
- `/orderDate`: Aligns with AP-003 (date range) but hot in recent orders, cold for historical

**Expected Distribution**: {X million distinct values, roughly even distribution}

### Document Types

#### {DocumentType1} (discriminator: `"type": "{value}"`)

**Source Tables**: {ORACLE_TABLE (columns embedded), ORACLE_CHILD_TABLE (embedded as array)}

**Estimated Size**: {avg bytes} per document | **Estimated Count**: {total documents} | **Estimated Container Size**: {GB}

**JSON Schema**:
```json
{
  "id": "GUID",
  "type": "order",
  "customerId": "value — partition key",
  "orderNumber": "string — unique business key",
  "orderDate": "ISO-8601 — source: ORDERS.ORDER_DATE",
  "totalAmount": "number — source: ORDERS.TOTAL",
  "customerName": "string — denormalized from CUSTOMERS.NAME (change feed updated)",
  "items": [
    {
      "itemId": "ULID",
      "productId": "string",
      "productName": "string — snapshot at order time (never updated)",
      "quantity": "number",
      "unitPrice": "number"
    }
  ],
  "shippingAddressId": "string — reference to addresses container",
  "status": "string — enum: pending | confirmed | shipped | delivered",
  "_metadata": {
    "createdAt": "ISO-8601",
    "updatedAt": "ISO-8601",
    "version": 1
  }
}
```

**Column Mapping** (Oracle → Cosmos DB):

| Oracle Table | Oracle Column | JSON Path | Type Conversion | Notes |
|--------------|---------------|-----------|-----------------|-------|
| ORDERS | ORDER_ID | legacyOrderId | NUMBER(12) → string | Preserved for traceability |
| ORDERS | ORDER_NUMBER | orderNumber | VARCHAR2(20) → string | Business key, unique |
| ORDERS | ORDER_DATE | orderDate | DATE → ISO-8601 | Sorted descending in queries |
| ORDERS | CUSTOMER_ID | customerId | NUMBER(12) → string | Partition key |
| ORDERS | TOTAL | totalAmount | NUMBER(10,2) → number | Calculation validated in app |
| ORDERS_ITEMS | ITEM_ID | items[].itemId | NUMBER(12) → ULID | Sortable ID |
| CUSTOMERS | NAME | customerName | VARCHAR2(100) → string | Denormalized, refreshed via change feed |

**Constraint Migration**:

| Oracle Constraint | Type | Cosmos DB Handling |
|------------------|------|-------------------|
| PK: ORDER_ID | Primary | `id` field (unique per partition) |
| UQ: ORDER_NUMBER | Unique | Unique key policy: `/orderNumber` |
| FK: CUSTOMER_ID → CUSTOMERS.ID | Foreign Key | App-layer validation in OrderService.create() |
| CK: TOTAL >= 0 | Check | App-layer validation before write |
| NOT NULL: ORDER_DATE | NOT NULL | App-layer validation (required field) |

**Trigger → Change Feed Mapping**:

| Oracle Trigger | Event | Cosmos DB Change Feed Processor | Azure Function | Target Action |
|---------------|-------|-------------------------------|----------------|--------------|
| TRG_ORDER_AUDIT | AFTER INSERT/UPDATE | orders-feed-processor | fn-order-audit | Write audit record to audit-events container |
| TRG_CUSTOMER_NAME_SYNC | (implied by denormalization) | customers-feed-processor | fn-order-update-customer-name | Update customerName in orders container |

**Indexing Policy**:
```json
{
  "indexingMode": "consistent",
  "automatic": true,
  "includedPaths": [
    {"path": "/*"}
  ],
  "excludedPaths": [
    {"path": "/items[*]/internalNotes/?"}
  ],
  "compositeIndexes": [
    [
      {"path": "/customerId", "order": "ascending"},
      {"path": "/orderDate", "order": "descending"}
    ],
    [
      {"path": "/status", "order": "ascending"},
      {"path": "/orderDate", "order": "descending"}
    ]
  ]
}
```

**Throughput Estimate**:
- **Reads**: AP-001 (point read by ID) 500/sec × 1 RU = 500 RU; AP-002 (range query) 100/sec × 5 RU = 500 RU → ~1000 RU/s
- **Writes**: New orders 100/sec × 8 RU = 800 RU; Updates 50/sec × 5 RU = 250 RU → ~1050 RU/s
- **Recommendation**: Autoscale 2000–5000 RU/s

---

{Repeat Container and Document Type sections for each container}

## Denormalization Registry

| Duplicated Data | Source Container | Embedded In Container | Update Strategy | Staleness Tolerance | Rationale |
|-----------------|------------------|----------------------|-----------------|-------------------|-----------|
| customerName | customers | orders | Change feed: customers-feed-processor writes to orders container when customer name changes | < 30 seconds | AP-004 needs customer name on order without separate lookup |
| productName, productPrice | products | order-items | Snapshot at order time; fields named `snapshotProductName`, `snapshotProductPrice` — never updated | N/A (intentional) | Order must show price paid at time of order, not current price |
| shippingAddress | addresses | orders | Snapshot at order time; stored in `shippingAddressSnapshot` object — never updated | N/A (intentional) | Legal requirement: preserve address as customer ordered |

## View → Query/Denormalization Mapping

For views referenced in `out_of_scope_refs`, map to Cosmos DB implementation without requiring full view bodies:

| Oracle View | Source Tables | Cosmos DB Strategy | Implementation | Notes |
|------------|---------------|-------------------|-----------------|-------|
| VW_RECENT_ORDERS | ORDERS, CUSTOMERS (inferred) | Denormalized query pattern | Query orders container WHERE customerId = ? ORDER BY orderDate DESC LIMIT 10 | View body not available; inferred from name |
| VW_ORDER_SUMMARY | ORDERS, ORDER_ITEMS (inferred) | Materialized denormalized documents | Change feed processor → summary container with aggregated totals | Logic TBD; may require app-layer aggregation |
| VW_ORDER_AGING | ORDERS (inferred) | Query pattern | Query orders container WHERE status IN ('pending','confirmed') and DATEDIFF(day, orderDate, NOW()) > 30 | View body not available; assumed a time-based filter |

## Sequence → ID Strategy

For sequences referenced in `out_of_scope_refs`, determine ID generation strategies without requiring full sequence definitions:

| Oracle Sequence | Used By Table(s) | New Strategy | Rationale | Legacy Migration | Notes |
|-----------------|------------------|-------------|-----------|-----------------|-------|
| ORDER_SEQ | ORDERS.ORDER_ID | GUID v4 | No coordination needed, no hot spots | Preserve as `legacyOrderId` during migration | From out_of_scope_refs |
| ITEM_SEQ | ORDER_ITEMS.ITEM_ID | ULID | Sortable by time, preserves insertion order | Not used in app — safe to replace | From out_of_scope_refs |
| CUST_ADDR_SEQ | ADDRESSES.ADDR_ID | GUID v4 | Simple, collision-free | Preserve as `legacyAddressId` during migration | From out_of_scope_refs (ADDRESSES may be missing) |

## Cross-Partition Query Catalog

These patterns require cross-partition queries (expensive — use sparingly or denormalize):

| Query | Why Cross-Partition | Frequency | Mitigation | RU Cost Estimate |
|-------|-------------------|-----------|-----------|-----------------|
| "All orders by date range (date >= ? AND date <= ?)" | Partition key is customerId, not date | Daily reporting (batch) | Denormalized time-partitioned orders-by-date container or Azure Synapse for analytics | ~10,000 RU for full scan + filter |
| "All pending orders globally" | Partition key is customerId, status is scattered | Rare admin (< 1/min) | Accept cross-partition cost OR materialized view in separate container filtered by status | ~5,000 RU (fewer hits) |

## Inventory Scope & Dependency Gaps

**Analysis Completeness**:
- [ ] Dependency gaps checked and documented (if any exist)
- [ ] Out-of-scope references (sequences, triggers, views) identified and addressed in design

**Provided Objects Coverage**:
- [ ] All provided tables mapped to containers/documents
- [ ] All provided PLSQL procedures/functions/packages mapped to services
- [ ] All provided constraints mapped to enforcement

**Referenced Objects (out_of_scope_refs) Coverage**:
- [ ] All referenced sequences mapped to ID strategies
- [ ] All referenced triggers mapped to change feed handlers with body availability notes
- [ ] All referenced views mapped to query/denormalization patterns
- [ ] Any other out-of-scope references (jobs, synonyms, types) noted for future consideration

## Summary & Sign-Off

- [ ] All access patterns from Phase 1 verified as satisfied by the document model
- [ ] All provided Oracle objects accounted for and mapped
- [ ] All referenced out-of-scope objects (triggers, sequences, views) addressed with design strategies
- [ ] Dependency gaps (if any) documented with notes on design impacts
- [ ] Every constraint migration documented and enforcement responsibility assigned
- [ ] Partition key distribution confirmed as reasonable
- [ ] Denormalization strategy documented with update plans
- [ ] Cross-partition queries identified and mitigated
- [ ] Ready for migration codegen (gen-migration)
```

### Output 2 — `cosmos-config.json`

Machine-readable Cosmos DB configuration for Azure deployment:

```json
{
  "database": "{database-name}",
  "containers": [
    {
      "name": "orders",
      "partitionKey": "/customerId",
      "uniqueKeys": [
        ["/orderNumber"]
      ],
      "defaultTtl": -1,
      "changeFeedPolicy": {
        "retentionDuration": "P7D"
      },
      "indexingPolicy": {
        "indexingMode": "consistent",
        "automatic": true,
        "includedPaths": [{"path": "/*"}],
        "excludedPaths": [{"path": "/items[*]/internalNotes/?"}],
        "compositeIndexes": [...]
      },
      "throughput": {
        "type": "autoscale",
        "maxRUs": 5000
      }
    }
  ],
  "changeFeedProcessors": [
    {
      "name": "orders-audit-processor",
      "sourceContainer": "orders",
      "leaseContainer": "leases",
      "processorHandler": "fn-order-audit",
      "oracleTriggerMapped": "TRG_ORDER_AUDIT",
      "targetContainer": "audit-events"
    }
  ]
}
```

### Output 3 — `access-patterns.md`

Catalog of every access pattern and how it's served:

```markdown
# Access Pattern Catalog: {schema-name}

| # | Pattern Name | Frequency | Oracle Implementation | Cosmos DB Implementation | Container | Partition Key Aligned? | RU Estimate | Notes |
|---|--------------|-----------|---------------------|------------------------|-----------|----------------------|-----------|-------|
| AP-001 | Get order by ID | High (user dashboard) | SELECT * FROM ORDERS WHERE ORDER_ID = ? | Point read by id + partition key (must know customerId) | orders | Yes | 1 RU | Requires knowing customerId; redesign if only have orderId |
| AP-002 | Get all orders for customer | High (order history) | SELECT * FROM ORDERS WHERE CUSTOMER_ID = ? ORDER BY ORDER_DATE DESC | Query within partition (single RU charge) | orders | Yes | 5-10 RU | Single-partition, efficient |
| AP-003 | Search orders by date range | Medium (reporting) | SELECT * FROM ORDERS WHERE ORDER_DATE BETWEEN ? AND ? | Cross-partition query ⚠ | orders | No | 50-100+ RU | Consider: materialized time-partitioned container or Azure Synapse |
| AP-004 | Get order + customer details | Medium | SELECT O.*, C.NAME FROM ORDERS O JOIN CUSTOMERS C... | Point read returns embedded customerName | orders | Yes | 1 RU | customerName denormalized, refreshed via change feed |
| AP-005 | Get order items | Medium (detail view) | SELECT * FROM ORDER_ITEMS WHERE ORDER_ID = ? | Embedded array in order document — no separate query | orders | Yes | 1 RU | Items embedded; no JOIN needed |
```

## Guidelines

1. **Access Patterns Drive Design** — every design decision traces back to a query pattern
2. **Avoid Hot Partitions** — uneven distribution kills throughput
3. **Minimize Cross-Partition Queries** — they're 10-100x more expensive; denormalize if frequent
4. **Denormalization is Intentional** — document every duplicate with update strategy
5. **Constraints Are Yours Now** — FKs, UQs, CHECKs all move to app layer; catalog them
6. **Triggers Become Change Feeds** — every trigger maps to a change feed processor
7. **Test Partition Distribution** — simulate write volume against partition key candidates
8. **Version Your Documents** — `_metadata.version` enables optimistic concurrency control
9. **Preserve Legacy IDs** — maintain Oracle IDs as `legacy*` fields for traceability during migration

## Output Checklist

- [ ] Pre-check complete: dependency_gaps warnings issued if analysis is incomplete
- [ ] document-model.md created with all containers, document types, schemas, constraint mappings, trigger mappings
- [ ] All **provided** Oracle tables mapped to Cosmos DB containers/documents (with rationale for embedding decisions)
- [ ] All columns from provided tables mapped to JSON properties with type conversions documented
- [ ] All constraints from provided DDL migrated to app layer with enforcement responsibility documented
- [ ] All **provided** PLSQL procedures/functions/packages mapped to app-layer services
- [ ] All **referenced triggers** (from out_of_scope_refs) mapped to change feed processors with note on body availability
- [ ] All **referenced sequences** (from out_of_scope_refs) mapped to ID strategies
- [ ] All **referenced views** (from out_of_scope_refs) mapped to denormalization patterns or query strategies
- [ ] Partition keys chosen with 2-3 alternatives evaluated and pros/cons documented
- [ ] Denormalization registry complete (duplicated field → source → embedded in → update strategy)
- [ ] Access patterns verified as satisfiable by document model
- [ ] Indexing policy specified for each container
- [ ] Throughput estimates provided (RU/s) with autoscale recommendations
- [ ] cosmos-config.json created with machine-readable container configs
- [ ] access-patterns.md created cataloging all patterns and how they're served
- [ ] Cross-partition queries identified and mitigation strategies documented
- [ ] Dependency gaps documented as requiring future design revisits once missing objects are analyzed
- [ ] Ready for migration generation (gen-migration skill)
