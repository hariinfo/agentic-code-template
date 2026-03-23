# Database Modernization Pipeline: Oracle → Azure Cosmos DB (NoSQL)

## Overview

A five-skill pipeline that takes an Oracle database schema through analysis → document model design → migration generation → validation, producing a fully designed Cosmos DB NoSQL data layer with migration scripts. This pipeline can run standalone OR consume artifacts from the code modernization pipeline when available.

### Why This Is Hard

Oracle → Cosmos DB is not a "port" — it's a **redesign**. Relational databases normalize data and use JOINs at query time. Cosmos DB denormalizes data and optimizes for access patterns at write time. Every fundamental assumption changes:

| Oracle (Relational) | Cosmos DB (NoSQL) | Impact |
|---------------------|-------------------|--------|
| Normalized tables with JOINs | Denormalized documents, embed or reference | Must redesign every table relationship |
| Foreign key constraints | No FK enforcement — app layer responsibility | Must catalog all constraints for app-layer migration |
| Stored procedures (PLSQL) | App layer (Spring Boot services) | PLSQL business logic moves to code pipeline |
| Complex SQL (subqueries, CTEs, window functions) | Limited SQL API, no JOINs across partitions | Must decompose complex queries into access-pattern-driven reads |
| Schema enforced by DB | Schema-free documents — app layer validates | Must capture all CHECK constraints, NOT NULL, data types |
| Single-row transactions across tables | Transactional batch within single partition key only | Must identify cross-table transactions and redesign |
| Sequences for IDs | No sequences — use GUIDs or app-generated IDs | Flag sequence references for ID strategy decisions |
| Triggers fire on DML | Change feed + Azure Functions | Flag trigger references for change feed mapping |

### Input Model — Controlled & Incremental

Unlike the code pipeline's analyze-entry (which auto-discovers files), the DB pipeline uses a **controlled input model**:

1. **User provides DDL** — pasted directly into prompts or uploaded as .sql files
2. **Scoped to 3 object types**: tables (with constraints), stored procedures/functions, packages
3. **Incremental** — run `/analyze-schema` multiple times, each time adding new objects. The skill appends to the existing inventory and re-evaluates gaps.
4. **Gap detection** — when a FK references a table you haven't provided yet, or PLSQL calls a procedure not yet given, it flags the gap so you know what to provide next.
5. **Out-of-scope flagging** — references to triggers, sequences, views, jobs, synonyms etc. are flagged for awareness but not expected as input.

### Pipeline Diagram

```
                         PER SCHEMA / MODULE
┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
│  analyze-schema   │──▶│  design-documents │──▶│  gen-migration    │──▶│ validate-migration│
│  (Step 1)         │   │  (Step 2)         │   │  (Step 3)         │   │ (Step 4)          │
└──────────────────┘   └──────────────────┘   └──────────────────┘   └──────────────────┘
       │                       │                       │                       │
       ▼                       ▼                       ▼                       ▼
 schema-analysis.md     document-model.md       migration/               validation-report.md
 oracle-inventory.json  cosmos-config.json      ├── adf-pipeline.json
 relationship-map.md    access-patterns.md      ├── transform-scripts/
                                                └── rollback-plan.md

                    OPTIONAL: READS FROM CODE PIPELINE
                    ┌─────────────────────────────────┐
                    │  modernization-output/{slug}/    │
                    │  ├── analysis.md                 │
                    │  ├── source-extracts.md          │
                    │  └── techspec.md                 │
                    └─────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│  db-modernize  (Orchestrator)                             │
│  Runs Steps 1-4 end-to-end for a schema/module            │
└──────────────────────────────────────────────────────────┘
```

---

## 1. Directory & File Conventions

All artifacts live under a **db-slug directory**:

```
modernization-output/
  db-{slug}/                          ← e.g., db-orders, db-payments, db-full-schema
    schema-analysis.md                ← Step 1: comprehensive Oracle schema analysis
    oracle-inventory.json             ← Step 1: machine-readable inventory of all DB objects
    relationship-map.md               ← Step 1: entity relationship documentation
    document-model.md                 ← Step 2: Cosmos DB document model design
    cosmos-config.json                ← Step 2: Cosmos DB configuration (containers, indexing, RU)
    access-patterns.md                ← Step 2: access pattern catalog driving the design
    migration/                        ← Step 3: all migration artifacts
      adf-pipeline.json              ← Azure Data Factory pipeline definition
      transform-scripts/             ← Per-table transformation scripts (Python/JS)
      seed-data/                     ← Reference/lookup data as JSON documents
      rollback-plan.md               ← Rollback strategy if migration fails
    validation-report.md             ← Step 4: completeness and correctness verification
```

The **db-slug** is derived from the schema or module name (e.g., `db-orders` for the ORDERS schema, `db-full-schema` for the entire database).

### Integration with Code Pipeline

When code modernization artifacts exist for the same module, the DB pipeline reads them for context:
- `source-extracts.md` → reveals which queries the app actually executes (vs. DB objects that exist but may be unused)
- `techspec.md` → reveals transaction boundaries, caching patterns, and access patterns from the app perspective
- `analysis.md` → reveals which tables/procedures are actually referenced from application code

This context helps the design-documents skill make better decisions about denormalization and partition key selection based on real access patterns rather than guessing.

---

## 2. Skill #1 — `analyze-schema`

### Purpose
Catalog Oracle database objects from user-provided DDL extracts — tables, constraints, and PLSQL code — and produce a comprehensive inventory. This is a **controlled, incremental** process: the user provides DDL for specific objects, and the skill accumulates them into a growing inventory over multiple runs.

### Input Model — Controlled & Incremental

Unlike the code pipeline's analyze-entry (which auto-discovers everything), analyze-schema is **user-driven**:

1. **The user provides DDL** — either pasted directly into the prompt or uploaded as .sql files
2. **Scope is limited to what the user provides**: tables, constraints, PLSQL (stored procedures, functions, packages)
3. **Incremental accumulation** — each run appends to the existing `oracle-inventory.json`. If the file already exists, the skill reads it first and merges new objects in. If a table is provided again, it overwrites the previous version (for corrections).
4. **Gap detection** — when the skill encounters a FK reference, PLSQL call, or table reference that hasn't been provided yet, it flags it as a dependency gap: "⚠ TABLE CUSTOMERS referenced by FK but DDL not yet provided"
5. **No auto-discovery** — the skill does NOT scan the filesystem or codebase for SQL files. It only processes what the user explicitly provides.

### Trigger
`/analyze-schema <db-slug>`

- **db-slug** — the module/schema name (e.g., `orders`, `payments`). Creates `modernization-output/db-{slug}/`
- The user then provides DDL in the same prompt (pasted or as uploaded files)

### Input
- **User-provided DDL** (required) — CREATE TABLE statements, constraint definitions, PLSQL code (procedures, functions, packages)
- **Existing inventory** (optional, auto-detected) — `modernization-output/db-{slug}/oracle-inventory.json` from previous runs
- **Code pipeline artifacts** (optional) — `modernization-output/{slug}/source-extracts.md` if the code pipeline has run for related entry points

### Object Types in Scope

| Object Type | What to Extract | Provided By User |
|------------|----------------|-----------------|
| **Tables** | Full DDL: columns, types, precision, nullable, defaults | ✅ Yes |
| **Constraints** | PK, FK (with ON DELETE/UPDATE), unique, check, NOT NULL | ✅ Yes (in CREATE TABLE or ALTER TABLE) |
| **Stored Procedures** | Full PLSQL body, parameters, local variables, SQL statements | ✅ Yes |
| **Functions** | Full PLSQL body, parameters, return type | ✅ Yes |
| **Packages** | Specification (public interface) + body (implementation) | ✅ Yes |
| Triggers | — | ❌ Out of scope (flag if referenced) |
| Sequences | — | ❌ Out of scope (flag if referenced in PLSQL or DDL) |
| Views | — | ❌ Out of scope (flag if referenced) |
| Jobs | — | ❌ Out of scope |
| Synonyms / DB Links | — | ❌ Out of scope (flag if referenced) |
| Grants | — | ❌ Out of scope |
| User-Defined Types | — | ❌ Out of scope (flag if referenced) |

For out-of-scope objects that are **referenced** by in-scope objects (e.g., a sequence used in a trigger referenced by a table, or a view referenced by PLSQL), the skill flags them as dependency gaps so the user knows what else may need attention.

### Analysis Logic

**Phase 1 — Parse & Catalog Provided DDL**

1. **Tables** — for each CREATE TABLE provided:
   - Full column inventory: name, data type (with precision, scale), nullable, default value
   - Primary keys (single and composite)
   - Foreign keys with referenced table/column and ON DELETE/ON UPDATE actions
   - Unique constraints (with constraint name)
   - Check constraints (capture the full expression)
   - NOT NULL constraints
   - Column-level and table-level comments (if included in DDL)
   - Indexes defined in the DDL (inline or CREATE INDEX)
   - Partitioning strategy if present (range, list, hash) with partition key columns
   - LOB columns (CLOB, BLOB) — flag for Azure Blob Storage consideration

2. **Constraints** — consolidated from all provided DDL:
   - Every constraint across all tables: name, type, columns, expression, deferrable, initially deferred
   - FK relationship graph: parent → child with cardinality indicators
   - Flag any FK that references a table NOT yet provided: "⚠ FK references TABLE_X but DDL not yet provided"

3. **Stored Procedures & Functions** — for each provided:
   - Full PLSQL body (preserved verbatim, not summarized)
   - Parameters: name, type, direction (IN/OUT/INOUT), default values
   - Local variables and cursors
   - All SQL statements executed (SELECT, INSERT, UPDATE, DELETE, MERGE) — extract each one
   - Tables touched: read set and write set
   - Exception handling blocks (WHEN clauses and their actions)
   - Transaction control (COMMIT, ROLLBACK, SAVEPOINT, autonomous transactions)
   - Calls to other procedures/functions (flag if that procedure hasn't been provided yet)
   - Dynamic SQL (EXECUTE IMMEDIATE) — capture the constructed SQL patterns
   - Bulk operations (FORALL, BULK COLLECT)
   - Cursor loops and their query patterns

4. **Packages** — for each provided:
   - Package specification (public interface): all procedure/function signatures, types, constants, exceptions, cursors
   - Package body: full implementation of each procedure/function (same detail as #3)
   - Package-level variables (state that persists across calls within a session)
   - Initialization block
   - Dependencies between package procedures

**Phase 2 — Dependency Gap Detection**

5. **Flag missing dependencies** — scan all provided DDL for references to objects NOT yet provided:
   - FK references to tables not yet in the inventory → "⚠ DEPENDENCY GAP: TABLE_X"
   - PLSQL calls to procedures/functions not yet provided → "⚠ DEPENDENCY GAP: PROC_Y"
   - References to sequences → "⚠ OUT OF SCOPE: SEQUENCE_Z (referenced by TABLE_X.COL)"
   - References to triggers → "⚠ OUT OF SCOPE: TRIGGER_T (referenced in DDL for TABLE_X)"
   - References to views → "⚠ OUT OF SCOPE: VIEW_V (referenced in PLSQL PROC_Y)"
   - References to synonyms, DB links → "⚠ OUT OF SCOPE: cross-schema dependency"
   - References to user-defined types → "⚠ OUT OF SCOPE: TYPE_T (used as column type or parameter)"

6. **Incremental merge** — if `oracle-inventory.json` already exists from a prior run:
   - Read existing inventory
   - Add new tables/procedures/packages
   - If a table is re-provided, overwrite the previous version (user is correcting)
   - Re-evaluate dependency gaps (a previously missing table may now be provided)
   - Update relationship map with newly discovered connections

**Phase 3 — Relationship Analysis**

7. **Build the entity relationship map** from provided DDL:
   - Direct FK relationships (parent → child) — from constraint definitions
   - Implicit relationships (columns named `{TABLE}_ID` without FK constraints) — flag as "possible relationship"
   - Join patterns from provided PLSQL (tables frequently joined together)
   - Circular references and self-referencing tables
   - Inheritance patterns (tables with discriminator columns suggesting type hierarchies)
   - Bridge/junction tables for many-to-many relationships
   - For each relationship: cardinality (1:1, 1:N, N:M), nullable, cascade behavior

8. **Identify data access patterns** from provided PLSQL:
   - Extract every query from procedures/functions/packages
   - Group by table: which tables are queried most, which are write-heavy
   - Identify common WHERE clause patterns → partition key candidates for Cosmos DB
   - If code pipeline artifacts available: cross-reference with `source-extracts.md` for app-level query patterns
   - Identify write patterns: single-row inserts, bulk inserts, update-heavy, append-only

**Phase 4 — Cosmos DB Readiness Assessment**

9. **Flag migration challenges** for each provided object:
   - Tables with 50+ columns → likely needs splitting into multiple document types
   - Tables with many FK references → needs denormalization strategy
   - N:M relationships → needs embedding decision
   - PLSQL with complex transaction logic (multi-table COMMIT/ROLLBACK) → needs app-layer redesign
   - PLSQL with cursor loops doing row-by-row processing → needs batch/bulk pattern redesign
   - LOB columns (CLOB/BLOB) → needs Azure Blob Storage integration
   - Sequence references → needs ID generation strategy (GUID, ULID, snowflake)
   - Cross-table constraints → app-layer enforcement in Cosmos DB

10. **Self-validation pass**:
    - Confirm every column in every provided table is captured
    - Confirm every parameter in every provided procedure is captured
    - Confirm all FK relationships are mapped (even to missing tables)
    - Report: "{N} tables, {M} procedures, {K} packages cataloged. {G} dependency gaps remaining."

### Output 1 — `schema-analysis.md`

```markdown
# Oracle Schema Analysis: {schema-name}

## Schema Overview
- **Schema**: {name}
- **Total Tables Provided**: {count}
- **Total Stored Procedures Provided**: {count}
- **Total Functions Provided**: {count}
- **Total Packages Provided**: {count} ({total procedures across packages})
- **Dependency Gaps**: {count} (referenced but not yet provided)
- **Out-of-Scope References**: {count} (sequences, triggers, views, etc.)
- **Run Mode**: {Initial | Incremental — added {N} tables, {M} procedures this run}

## ⚠ Dependency Gaps (Provide DDL for These Next)
| # | Object | Type | Referenced By | Context |
|---|--------|------|--------------|---------|
| 1 | CUSTOMERS | Table | ORDERS.CUSTOMER_ID (FK) | Foreign key reference |
| 2 | PROC_VALIDATE_ORDER | Procedure | PKG_ORDER_MGMT body | Called but not provided |
| ... | ... | ... | ... | ... |

## ⚠ Out-of-Scope References (Not Expected — For Awareness)
| # | Object | Type | Referenced By | Notes |
|---|--------|------|--------------|-------|
| 1 | ORDER_SEQ | Sequence | ORDERS table (DEFAULT) | ID generation — will need GUID strategy |
| 2 | TRG_ORDER_AUDIT | Trigger | ORDERS table (DDL comment) | Business logic — needs change feed mapping |
| ... | ... | ... | ... | ... |

## Tables

### {SCHEMA.TABLE_NAME}
**Columns**: {count} | **Constraints**: {count}
**Purpose**: {inferred from name, columns, comments, PLSQL usage}

| # | Column | Type | Nullable | Default | Constraints | Notes |
|---|--------|------|----------|---------|-------------|-------|
| 1 | ORDER_ID | NUMBER(12) | NOT NULL | — | PK | Primary key |
| 2 | CUSTOMER_ID | NUMBER(12) | NOT NULL | — | FK → CUSTOMERS.ID (ON DELETE CASCADE) | ⚠ Target table not yet provided |
| ... | ... | ... | ... | ... | ... | ... |

**Indexes** (from DDL):
| Name | Columns | Type | Unique |
|------|---------|------|--------|
| IDX_ORDER_CUSTOMER | CUSTOMER_ID | BTREE | No |

**Referenced By**: ORDER_ITEMS (FK), PAYMENTS (FK)
**References**: CUSTOMERS (FK ⚠ not provided), PRODUCTS (implicit join in PLSQL)

**Cosmos DB Migration Notes**:
- ⚠ {count} foreign keys reference this table — embedding decision required
- Partition key candidate: CUSTOMER_ID (most common WHERE clause in PLSQL)

(Repeat for every provided table)

## Stored Procedures & Functions

### {PROCEDURE_NAME}
**Parameters**: {list with types and directions}
**Tables Read**: {list}
**Tables Written**: {list}
**Transaction Control**: {COMMIT/ROLLBACK/SAVEPOINT details}
**Migration Strategy**: Moves to Spring Boot service layer

Full PLSQL Body:
(preserved verbatim)

**SQL Statements Extracted**:
| # | Type | Table(s) | Purpose |
|---|------|----------|---------|
| 1 | SELECT | ORDERS, ORDER_ITEMS | Fetch order with items |
| 2 | UPDATE | ORDERS | Update status |
| ... | ... | ... | ... |

(Repeat for every provided procedure/function)

## Packages

### {PACKAGE_NAME}
**Specification**: (public interface — signatures, types, constants)
**Body**: (full implementation per procedure, same detail as above)
**Package-Level State**: {variables that persist across calls}

(Repeat for every provided package)

## Migration Complexity Assessment
| Category | Count | Complexity | Notes |
|----------|-------|-----------|-------|
| Simple tables (< 10 cols, no complex constraints) | {N} | Low | Direct document mapping |
| Complex tables (many FKs, LOBs) | {N} | High | Requires design decisions |
| PLSQL with business logic | {N} | High | Moves to app layer |
| PLSQL with multi-table transactions | {N} | Critical | Cosmos DB single-partition transactions only |
| Dependency gaps remaining | {N} | — | Provide DDL to complete analysis |
| Total estimated effort | — | {S/M/L/XL} | — |
```

### Output 2 — `oracle-inventory.json`

Machine-readable inventory that accumulates across incremental runs:

```json
{
  "schema": "APP",
  "generated_at": "2026-03-22T15:00:00Z",
  "last_updated": "2026-03-22T16:30:00Z",
  "run_history": [
    {"timestamp": "2026-03-22T15:00:00Z", "added": ["ORDERS", "ORDER_ITEMS"], "mode": "initial"},
    {"timestamp": "2026-03-22T16:30:00Z", "added": ["CUSTOMERS", "PKG_ORDER_MGMT"], "mode": "incremental"}
  ],
  "tables": {
    "ORDERS": {
      "columns": [
        {"name": "ORDER_ID", "type": "NUMBER(12)", "nullable": false, "pk": true},
        {"name": "CUSTOMER_ID", "type": "NUMBER(12)", "nullable": false, "fk": {"table": "CUSTOMERS", "column": "ID", "on_delete": "CASCADE"}}
      ],
      "constraints": [...],
      "indexes": [...],
      "referenced_by": ["ORDER_ITEMS", "PAYMENTS"],
      "references": ["CUSTOMERS"]
    }
  },
  "procedures": {
    "PROC_CREATE_ORDER": {
      "parameters": [...],
      "tables_read": ["ORDERS", "CUSTOMERS"],
      "tables_written": ["ORDERS", "ORDER_HISTORY"],
      "calls": ["PROC_VALIDATE_ORDER"],
      "has_transaction_control": true
    }
  },
  "packages": {...},
  "dependency_gaps": [
    {"object": "CUSTOMERS", "type": "table", "referenced_by": ["ORDERS.CUSTOMER_ID"]},
    {"object": "PROC_VALIDATE_ORDER", "type": "procedure", "referenced_by": ["PROC_CREATE_ORDER"]}
  ],
  "out_of_scope_refs": [
    {"object": "ORDER_SEQ", "type": "sequence", "referenced_by": ["ORDERS"]},
    {"object": "TRG_ORDER_AUDIT", "type": "trigger", "referenced_by": ["ORDERS"]}
  ]
}
```

### Output 3 — `relationship-map.md`

Entity relationship documentation with Mermaid diagrams:

```markdown
# Entity Relationship Map: {schema-name}

## ER Diagram
{Mermaid erDiagram showing all tables and relationships}

## Relationship Catalog
| Parent | Child | Cardinality | FK Column | Cascade | Nullable |
|--------|-------|-------------|-----------|---------|----------|
| CUSTOMERS | ORDERS | 1:N | CUSTOMER_ID | CASCADE | No |
| ORDERS | ORDER_ITEMS | 1:N | ORDER_ID | CASCADE | No |
| ... | ... | ... | ... | ... | ... |

## Aggregate Boundaries (Suggested)
Based on relationship patterns, the following natural aggregates emerge:
1. **Order Aggregate**: ORDERS + ORDER_ITEMS + ORDER_HISTORY (parent-child cascade)
2. **Customer Aggregate**: CUSTOMERS + ADDRESSES + PREFERENCES
3. **Product Catalog**: PRODUCTS + CATEGORIES + PRICING

These aggregates inform the Cosmos DB document model — each aggregate naturally becomes one or more Cosmos DB containers.
```

---

## 3. Skill #2 — `design-documents`

### Purpose
Take the Oracle schema analysis and design the target Cosmos DB document model — containers, document structures, partition keys, indexing policies, and denormalization strategy. This is the most intellectually challenging step: transforming a normalized relational model into an optimized NoSQL document model.

### Trigger
`/design-documents <db-slug>`

### Input
- `modernization-output/{db-slug}/schema-analysis.md` (required)
- `modernization-output/{db-slug}/oracle-inventory.json` (required)
- `modernization-output/{db-slug}/relationship-map.md` (required)
- Optionally: `modernization-output/{slug}/source-extracts.md` (from code pipeline — reveals actual query patterns)
- Optionally: `modernization-output/{slug}/techspec.md` (from code pipeline — reveals transaction boundaries)

### Design Logic

**Phase 1 — Identify Access Patterns**

Before designing documents, catalog every way the data is accessed. This drives all design decisions.

1. **From PLSQL/SQL analysis**: Extract every SELECT, INSERT, UPDATE, DELETE from stored procedures and views
2. **From code pipeline** (if available): Extract every query from source-extracts.md
3. **From indexes**: Existing indexes reveal the queries the DBA optimized for
4. **From views**: Views reveal common read patterns

For each access pattern, document:
- Operation type (read/write/update/delete)
- Frequency (real-time, batch, rare)
- Tables involved (which JOINs)
- Filter conditions (WHERE clauses → partition key candidates)
- Sort requirements (ORDER BY → index design)
- Volume (how many rows returned/affected)
- Latency requirement (interactive user-facing vs. background batch)

**Phase 2 — Design Document Model**

Apply NoSQL design principles to transform the relational model:

5. **Determine aggregate boundaries**:
   - Group tables that are always accessed together into single documents
   - Parent-child relationships with CASCADE delete → strong candidates for embedding
   - 1:1 relationships → almost always embed
   - 1:N where N is small (< 100) and child is always read with parent → embed
   - 1:N where N is large or child is accessed independently → separate container with reference
   - N:M relationships → duplicate data on both sides or use a reference pattern

6. **Choose containers** (Cosmos DB's equivalent of tables):
   - Each aggregate boundary typically becomes one container
   - Consider the "single container with discriminator" pattern for related entities that share the same partition key
   - Target: minimize the number of containers while maintaining clear access patterns

7. **Choose partition keys** — the most critical Cosmos DB decision:
   - Must be a property that exists on every document in the container
   - Should distribute data evenly (avoid hot partitions)
   - Should align with the most common query filter (queries within one partition are fast; cross-partition queries are expensive)
   - Candidates: customer_id, tenant_id, order_date (year-month), region, category
   - For each container, evaluate 2-3 candidates with pros/cons
   - Consider hierarchical partition keys (new in Cosmos DB) for multi-tenant scenarios

8. **Design document schemas**:
   - For each container, define the JSON document structure
   - Map Oracle columns → JSON properties (with type conversions)
   - Embed child entities as nested arrays/objects where appropriate
   - Add discriminator field (`type`: "order", "orderItem") if mixing entity types
   - Add partition key field if it's synthetic (not a natural business key)
   - Handle Oracle-specific types:
     - NUMBER → number or string (for precision > 15 digits)
     - DATE/TIMESTAMP → ISO-8601 string
     - CLOB → string (or Azure Blob Storage reference if > 2MB)
     - BLOB → Azure Blob Storage reference
     - RAW → base64 string
     - User-defined types → nested objects

9. **Handle denormalization**:
   - Identify which data gets duplicated (e.g., customer name embedded in order document)
   - For each duplication: document the update propagation strategy (change feed, eventual consistency, acceptable staleness)
   - Flag data that changes frequently — may be better as a reference than embedded copy

10. **Handle sequences / ID generation**:
    - Oracle sequences → GUID (default) or app-generated IDs (snowflake, ULID)
    - For each entity, recommend an ID strategy
    - If legacy code relies on sequential IDs for ordering, document the alternative (timestamp field, sort key)

11. **Handle constraints that disappear**:
    - Foreign keys → application-layer validation + referential integrity checks
    - Unique constraints → unique key policy in Cosmos DB (within partition) or app-layer check (cross-partition)
    - Check constraints → application-layer validation
    - NOT NULL → application-layer validation (Cosmos DB is schema-free)
    - Document every constraint and where enforcement moves to

12. **Handle triggers**:
    - Map each Oracle trigger to its Cosmos DB equivalent:
      - Audit triggers → Change feed + Azure Function that writes to audit container
      - Computed column triggers → App-layer computation before write
      - Cascade update triggers → Change feed + Azure Function for propagation
      - Validation triggers → App-layer validation before write
    - For each trigger, specify: the change feed processor, the function logic, the target container

13. **Design indexing policy**:
    - Default: Cosmos DB indexes everything (good for most cases)
    - For large documents: exclude paths that are never queried
    - Composite indexes for ORDER BY on multiple fields
    - Spatial indexes if geo queries needed
    - Per-container indexing policy JSON

14. **Estimate throughput (RU/s)**:
    - For each container: estimate read/write RU consumption based on document size and access patterns
    - Recommend provisioned vs. serverless vs. autoscale
    - Flag containers that may need dedicated throughput

**Phase 3 — Validate Design**

15. **Verify every access pattern is satisfied**:
    - For each access pattern from Phase 1, confirm it can be served by the document model
    - Identify patterns that require cross-partition queries (expensive — may need denormalization)
    - Identify patterns that require multiple queries (no JOINs — may need further embedding)

16. **Verify every Oracle object is accounted for**:
    - Every table → mapped to a container/document
    - Every column → mapped to a JSON property or explicitly excluded with reason
    - Every constraint → mapped to Cosmos DB feature or app-layer enforcement
    - Every trigger → mapped to change feed handler or app-layer logic
    - Every view → mapped to denormalized document or query pattern
    - Every sequence → mapped to ID strategy

### Output 1 — `document-model.md`

```markdown
# Cosmos DB Document Model: {schema-name}

## Design Decisions Summary
- **Containers**: {count}
- **Document Types**: {count}
- **Embedded Relationships**: {count}
- **Reference Relationships**: {count}
- **Denormalized Fields**: {count}
- **Constraints Moved to App Layer**: {count}
- **Triggers Converted to Change Feed**: {count}

## Container: {container-name}

### Partition Key: `/{partition-key-path}`
**Rationale**: {why this key was chosen — based on access patterns}
**Alternatives Considered**: {other candidates and why they were rejected}
**Distribution**: {expected cardinality and distribution}

### Document Type: {type-name} (discriminator: `"type": "{value}"`)
**Source Tables**: {ORACLE_TABLE_1, ORACLE_TABLE_2 (embedded)}
**Estimated Document Size**: {avg bytes}
**Estimated Count**: {total documents}

```json
{
  "id": "GUID",
  "type": "{discriminator}",
  "{partitionKey}": "{value}",
  "field1": "type — source: TABLE.COLUMN",
  "field2": "type — source: TABLE.COLUMN",
  "embeddedChild": [
    {
      "childField1": "type — source: CHILD_TABLE.COLUMN",
      "childField2": "type — source: CHILD_TABLE.COLUMN"
    }
  ],
  "referencedEntity": "{container}/{id} — source: TABLE.FK_COLUMN",
  "_metadata": {
    "createdAt": "ISO-8601",
    "updatedAt": "ISO-8601",
    "version": "integer (optimistic concurrency)"
  }
}
```

### Column Mapping
| Oracle Column | JSON Path | Type Conversion | Notes |
|--------------|-----------|----------------|-------|
| ORDERS.ORDER_ID | id | NUMBER → GUID | Legacy ID preserved in `legacyId` field |
| ORDERS.ORDER_DATE | orderDate | DATE → ISO-8601 string | — |
| ... | ... | ... | ... |

### Constraint Migration
| Oracle Constraint | Type | Cosmos DB Handling |
|------------------|------|-------------------|
| PK: ORDER_ID | Primary Key | `id` field (unique per partition) |
| FK: CUSTOMER_ID → CUSTOMERS.ID | Foreign Key | App-layer validation |
| UQ: ORDER_NUMBER | Unique | Unique key policy: `/orderNumber` |
| CK: TOTAL > 0 | Check | App-layer validation in OrderService |

### Indexing Policy
```json
{
  "indexingMode": "consistent",
  "includedPaths": [...],
  "excludedPaths": [...],
  "compositeIndexes": [...]
}
```

### Throughput Estimate
- **Reads**: ~{N} RU/s ({pattern description})
- **Writes**: ~{N} RU/s ({pattern description})
- **Recommendation**: {Autoscale {min}-{max} RU/s | Serverless | Provisioned {N} RU/s}

(Repeat for each container and document type)

## Trigger → Change Feed Mapping
| Oracle Trigger | Change Feed Processor | Azure Function | Target Action |
|---------------|----------------------|----------------|--------------|
| TRG_ORDER_AUDIT | orders-container | fn-order-audit | Write to audit-container |
| TRG_CALC_TOTAL | order-items-container | fn-calc-total | Update parent order document |

## View → Query/Denormalization Mapping
| Oracle View | Strategy | Implementation |
|------------|----------|---------------|
| VW_ORDER_SUMMARY | Denormalized document | Materialized via change feed into summary-container |
| VW_CUSTOMER_ORDERS | Query pattern | Query orders-container by partition key (customerId) |

## Sequence → ID Strategy
| Oracle Sequence | New Strategy | Notes |
|----------------|-------------|-------|
| ORDER_SEQ | GUID (default) | Legacy ORDER_ID preserved as `legacyOrderId` for migration traceability |
| ITEM_SEQ | ULID | Sortable, time-ordered — preserves insertion order |

## Denormalization Registry
| Duplicated Data | Source Container | Embedded In | Update Strategy | Staleness Tolerance |
|----------------|-----------------|-------------|----------------|-------------------|
| Customer name | customers | orders | Change feed propagation | < 1 minute |
| Product name/price | products | order-items | Snapshot at order time (never updated) | N/A — intentional |

## Cross-Partition Query Catalog
| Query | Why Cross-Partition | Frequency | Mitigation |
|-------|-------------------|-----------|-----------|
| "All orders by date range" | Partition key is customerId | Daily reporting | Materialized view via change feed OR separate time-partitioned container |
```

### Output 2 — `cosmos-config.json`

Machine-readable Cosmos DB configuration:

```json
{
  "database": "{database-name}",
  "containers": [
    {
      "name": "{container-name}",
      "partitionKey": "/{path}",
      "uniqueKeys": [["/orderNumber"]],
      "indexingPolicy": {...},
      "throughput": {"type": "autoscale", "maxRUs": 4000},
      "defaultTtl": -1,
      "changeFeedPolicy": {"retentionDuration": "P7D"}
    }
  ],
  "changeFeedProcessors": [
    {
      "name": "{processor-name}",
      "sourceContainer": "{container}",
      "leaseContainer": "leases",
      "handler": "{azure-function-name}",
      "triggerMapped": "{oracle-trigger-name}"
    }
  ]
}
```

### Output 3 — `access-patterns.md`

Catalog of every access pattern and how it's served:

```markdown
# Access Pattern Catalog

| # | Pattern | Frequency | Oracle Implementation | Cosmos DB Implementation | Container | Partition Key Used? | Estimated RU |
|---|---------|-----------|---------------------|------------------------|-----------|-------------------|------------|
| AP-001 | Get order by ID | High | SELECT * FROM ORDERS WHERE ID = ? | Point read by id + partition key | orders | Yes | 1 RU |
| AP-002 | Get all orders for customer | High | SELECT * FROM ORDERS WHERE CUSTOMER_ID = ? ORDER BY ORDER_DATE DESC | Query within partition | orders | Yes (customerId) | ~5 RU |
| AP-003 | Search orders by date range | Medium | SELECT * FROM ORDERS WHERE ORDER_DATE BETWEEN ? AND ? | Cross-partition query | orders | No ⚠ | ~50 RU |
| ... | ... | ... | ... | ... | ... | ... | ... |
```

---

## 4. Skill #3 — `gen-migration`

### Purpose
Generate the actual migration artifacts — data transformation scripts, Azure Data Factory pipeline definitions, seed data, and rollback plan. This turns the document model design into executable migration code.

### Trigger
`/gen-migration <db-slug>`

### Input
- `modernization-output/{db-slug}/schema-analysis.md`
- `modernization-output/{db-slug}/oracle-inventory.json`
- `modernization-output/{db-slug}/document-model.md`
- `modernization-output/{db-slug}/cosmos-config.json`
- `modernization-output/{db-slug}/access-patterns.md`

### Generation Logic

**Phase 1 — Infrastructure Setup**
1. Generate Cosmos DB provisioning script (Bicep/ARM template or Terraform):
   - Database creation
   - Container creation with partition keys, indexing policies, throughput settings
   - Unique key policies
   - Change feed lease containers
2. Generate Azure Function project for change feed processors:
   - One function per trigger-to-change-feed mapping from document-model.md
   - Proper Cosmos DB trigger bindings
   - Error handling and dead-letter patterns

**Phase 2 — Data Transformation Scripts**
For each Oracle table → Cosmos DB container mapping:
3. Generate a transformation script (Python or JavaScript) that:
   - Reads rows from the Oracle source (via CSV export or direct connection)
   - Applies type conversions (Oracle types → JSON-compatible types)
   - Embeds child records into parent documents (per document model)
   - Generates new IDs (GUID/ULID) while preserving legacy IDs in a `legacyId` field
   - Handles NULL values appropriately
   - Adds metadata fields (`_metadata.createdAt`, `_metadata.migratedAt`, `_metadata.version`)
   - Adds discriminator field if container holds multiple document types
   - Handles denormalization (copies referenced data into documents per the denormalization registry)
   - Outputs JSON documents ready for Cosmos DB bulk import

4. Generate Oracle export queries:
   - One SELECT per table, ordered by primary key for deterministic export
   - Include JOINs where child data needs to be embedded
   - Handle LOB columns (export to Azure Blob Storage separately)

**Phase 3 — Azure Data Factory Pipeline**
5. Generate ADF pipeline definition (JSON):
   - Source: Oracle linked service (or CSV/Parquet files in Azure Blob)
   - Transformation: Data Flow with the mapping logic
   - Sink: Cosmos DB linked service
   - Batching: configure bulk import batch sizes
   - Error handling: log failed documents, continue processing
   - Parallelism: partition data flow by natural boundaries

**Phase 4 — Validation Queries**
6. Generate pre-migration validation:
   - Row counts per Oracle table
   - Checksum/hash queries for data integrity verification
   - Sample record queries (first 10, last 10, random 10 per table)
7. Generate post-migration validation:
   - Document counts per Cosmos DB container
   - Sample document verification (compare against Oracle source)
   - Constraint verification (unique keys, not-null fields)
   - Relationship integrity (referenced documents exist)

**Phase 5 — Rollback Plan**
8. Generate rollback strategy:
   - Point-in-time restore configuration for Cosmos DB
   - Data preservation during migration (Oracle remains read-only, not deleted)
   - Blue-green cutover plan: app reads from Oracle, writes to both, then cuts over
   - Feature flag approach: toggle between Oracle and Cosmos DB data sources

### Output — `migration/`

```
migration/
├── infrastructure/
│   ├── cosmos-provision.bicep          ← Cosmos DB provisioning (Bicep template)
│   ├── cosmos-provision.tf             ← Alternative: Terraform
│   └── azure-functions/               ← Change feed processor Functions project
│       ├── host.json
│       ├── package.json
│       └── functions/
│           ├── fn-order-audit.js       ← Per-trigger function
│           └── fn-calc-total.js
├── transform-scripts/
│   ├── transform-orders.py            ← Per-table transformation
│   ├── transform-customers.py
│   ├── transform-products.py
│   └── common/
│       ├── type-converters.py         ← Oracle → JSON type conversion utilities
│       ├── id-generator.py            ← GUID/ULID generation with legacy ID preservation
│       └── denormalize.py             ← Denormalization helpers
├── export-queries/
│   ├── export-orders.sql              ← Oracle SELECT for extraction
│   ├── export-customers.sql
│   └── export-products.sql
├── adf-pipeline.json                  ← Azure Data Factory pipeline definition
├── validation/
│   ├── pre-migration-checks.sql       ← Oracle-side validation queries
│   ├── post-migration-checks.py       ← Cosmos DB-side validation script
│   └── reconciliation-report.py       ← Compare Oracle vs Cosmos DB
├── seed-data/
│   └── lookup-tables.json             ← Reference/static data as JSON
└── rollback-plan.md                   ← Rollback and cutover strategy
```

---

## 5. Skill #4 — `validate-migration`

### Purpose
After migration has been executed (or as a dry-run check), verify that the Cosmos DB data is complete, correct, and consistent with the Oracle source. Produces a validation report.

### Trigger
`/validate-migration <db-slug>`

### Input
- `modernization-output/{db-slug}/schema-analysis.md`
- `modernization-output/{db-slug}/oracle-inventory.json`
- `modernization-output/{db-slug}/document-model.md`
- `modernization-output/{db-slug}/cosmos-config.json`
- `modernization-output/{db-slug}/migration/` (the migration artifacts)

### Verification Logic

**Check 1 — Schema Completeness**
For every Oracle table, column, and constraint:
- Verify it appears in the document model (mapped to a container/property)
- Verify the type conversion is correct
- Verify constraints are either enforced by Cosmos DB features or documented as app-layer responsibility
- Flag any Oracle column with no corresponding JSON path

**Check 2 — Data Integrity (post-migration)**
- Compare row counts: Oracle table rows vs Cosmos DB document counts
- Sample verification: for N random records, compare field-by-field between Oracle and Cosmos DB
- Verify embedded child documents: count children in Oracle vs count embedded in parent
- Verify denormalized data: compare duplicated fields against source of truth

**Check 3 — Access Pattern Validation**
For each access pattern from access-patterns.md:
- Verify the Cosmos DB query executes successfully
- Verify it returns expected results (compare against Oracle equivalent)
- Measure RU consumption and compare against estimates

**Check 4 — Change Feed Processor Validation**
- Verify each Azure Function deploys and connects to the correct container
- Verify trigger-equivalent behavior: perform a write and confirm the change feed processor fires and produces the correct side effect

**Check 5 — Edge Cases**
- NULL handling: verify NULLs are preserved correctly (Oracle NULL → JSON missing/null)
- Empty strings vs NULL: Oracle treats empty string as NULL; verify this is handled
- Numeric precision: verify NUMBER(38,10) values survive conversion
- Date/timestamp: verify timezone handling
- LOB data: verify large text/binary references resolve correctly
- Unicode: verify multi-byte characters survive conversion

### Output — `validation-report.md`

```markdown
# Migration Validation Report: {db-slug}

## Summary
- **Oracle Objects Mapped**: {count}/{total} ({%})
- **Column Mappings Verified**: {count}/{total}
- **Access Patterns Verified**: {count}/{total}
- **Change Feed Processors Verified**: {count}/{total}
- **Issues Found**: {count}

## Overall Status: {🟢 PASS | 🟡 WARNINGS | 🔴 FAILURES}

## Schema Completeness
| Oracle Object | Mapped To | Status | Notes |
|--------------|-----------|--------|-------|
| ORDERS table | orders container | ✅ Complete | 18/18 columns mapped |
| ORDERS.TOTAL > 0 constraint | App-layer validation | ⚠ Verify | Must confirm OrderService validates |

## Data Integrity (if post-migration)
| Oracle Table | Oracle Rows | Cosmos Docs | Match | Notes |
|-------------|-------------|-------------|-------|-------|
| ORDERS | 5,000,000 | 5,000,000 | ✅ | — |
| ORDER_ITEMS | 15,000,000 | (embedded) | ✅ | Verified: avg 3 items per order |

## Access Pattern Performance
| Pattern | Expected RU | Actual RU | Status | Notes |
|---------|-----------|-----------|--------|-------|
| AP-001: Get order by ID | 1 RU | 1.2 RU | ✅ | — |
| AP-003: Date range query | ~50 RU | 280 RU | ⚠ | Cross-partition fan-out — consider materialized view |

## Issues & Recommendations
| # | Severity | Issue | Recommendation |
|---|----------|-------|---------------|
| 1 | 🔴 Critical | ... | ... |
| 2 | 🟡 Warning | ... | ... |
```

---

## 6. Orchestrator — `db-modernize`

### Trigger
`/db-modernize <db-slug>`

### Workflow — Incremental by Design

Because analyze-schema is user-driven and incremental, the orchestrator supports two modes:

**Mode 1: Analyze Only (incremental — run multiple times)**
```
/analyze-schema <db-slug>
(user provides DDL for a batch of tables/procedures)
→ Appends to oracle-inventory.json
→ Updates schema-analysis.md and relationship-map.md
→ Reports dependency gaps: "Provide DDL for these next: TABLE_X, PROC_Y"

(repeat until dependency gaps are resolved or accepted)
```

**Mode 2: Full Pipeline (run when analysis is complete)**
```
/db-modernize <db-slug>

1. Verify modernization-output/db-{slug}/ exists with analysis artifacts
2. Check dependency gaps in oracle-inventory.json — warn if any remain
3. Check if code pipeline artifacts exist for related slugs (optional context)
4. Run design-documents → document-model.md + cosmos-config.json + access-patterns.md
5. Run gen-migration → migration/ directory with all artifacts
6. Run validate-migration → validation-report.md
7. Report completion with links to all artifacts and any remaining gaps
```

### Typical User Workflow
```
1. /analyze-schema orders       ← provide ORDERS, ORDER_ITEMS DDL
   → "⚠ Gap: CUSTOMERS table referenced by FK"
2. /analyze-schema orders       ← provide CUSTOMERS DDL + PKG_ORDER_MGMT
   → "⚠ Gap: PROC_VALIDATE_ORDER called by package"
3. /analyze-schema orders       ← provide PROC_VALIDATE_ORDER
   → "✅ No dependency gaps remaining"
4. /db-modernize orders         ← runs Steps 2-4 (design → migration → validation)
```

---

## 7. Integration with Code Modernization Pipeline

When both pipelines are used together, they complement each other:

```
CODE PIPELINE                          DB PIPELINE
─────────────                          ───────────
analyze-entry ────────────────────────▶ analyze-schema reads
  source-extracts.md (queries)           actual app query patterns
  techspec.md (transactions)             transaction boundaries

                                       design-documents ─────────▶ gen-code reads
                                         document-model.md          Cosmos DB model
                                         cosmos-config.json         for Spring Data setup

gen-code ◀─────────────────────────────── document-model.md
  Generates Spring Data Cosmos DB          (replaces JPA entities
   repository classes                       with Cosmos DB entities)
```

### When Running Both Pipelines:
1. Run `/modernize` first (code pipeline) — this produces source-extracts.md with actual query patterns
2. Run `/db-modernize` second — it reads the code pipeline artifacts for better access pattern analysis
3. After document model is designed, re-run `/gen-code` with awareness of the Cosmos DB model (it should generate Spring Data for Azure Cosmos DB instead of Spring Data JPA)

### Code Generation Adjustments for Cosmos DB:
When document-model.md exists, the gen-code skill should:
- Use `spring-boot-starter-data-cosmosdb` instead of `spring-boot-starter-data-jpa`
- Generate `@Container` entities instead of `@Entity` JPA entities
- Generate Cosmos DB repository interfaces extending `CosmosRepository`
- Configure partition key annotations
- Remove Flyway migrations (no relational schema to manage)
- Add Azure Cosmos DB connection configuration
- Generate change feed processor configurations
