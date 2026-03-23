---
name: schema-analyzer
description: |
  Oracle database schema analyzer — controlled, user-driven catalog of tables, stored procedures,
  functions, and packages. User provides DDL directly (pasted or uploaded .sql files). Parses all
  DDL, catalogs objects incrementally into oracle-inventory.json, flags dependency gaps (FK and
  procedure references to objects not yet provided), identifies out-of-scope references (sequences,
  triggers, views), analyzes relationships and access patterns, and assesses Cosmos DB migration
  readiness. Produces schema-analysis.md, oracle-inventory.json, and relationship-map.md.
---

# Schema Analyzer Skill (analyze-schema)

## Purpose

Catalog Oracle database objects from user-provided DDL extracts and produce a comprehensive,
incrementally-accumulated inventory with dependency gap detection and migration readiness assessment.
This is a controlled, user-driven process: the user provides DDL for specific objects (tables,
constraints, stored procedures, functions, packages), and the skill catalogs them into a growing
inventory over multiple runs.

Unlike code-analysis (which auto-discovers all connected components), schema-analyzer is explicitly
scoped to what the user provides and maintains awareness of missing dependencies.

## Trigger Keywords

Oracle, database, schema, DDL, PLSQL, table, constraint, stored procedure, analyze schema,
database analysis, Oracle inventory, CREATE TABLE, CREATE PROCEDURE, CREATE PACKAGE, catalog tables,
inventory schema

## Input Model — Controlled & Incremental

**No auto-discovery** — the skill does NOT scan the filesystem or database. It processes only what
the user explicitly provides.

1. **User provides DDL** — pasted directly into the prompt or uploaded as .sql files
2. **Scope is limited to what is provided**: tables, constraints, PLSQL (stored procedures, functions, packages)
3. **Incremental accumulation** — each run appends to existing `oracle-inventory.json`. If the file already exists, the skill reads it first and merges new objects in. If a table is provided again, it overwrites the previous version (for corrections).
4. **Gap detection** — when the skill encounters a FK reference, PLSQL call, or table reference that hasn't been provided yet, it flags it as a dependency gap: "⚠ TABLE CUSTOMERS referenced by FK but DDL not yet provided"
5. **Out-of-scope awareness** — references to triggers, sequences, views, jobs, synonyms, types are flagged for awareness but not expected as input

## Input Specification

### Trigger Format

```
/analyze-schema <db-slug>
<paste DDL or attach .sql files>
```

- **db-slug** — the module/schema name (e.g., `orders`, `payments`, `full-schema`). Creates `modernization-output/db-{slug}/`
- User provides DDL in the same prompt (pasted or as uploaded files)

### Input Artifacts

- **User-provided DDL** (required) — CREATE TABLE, ALTER TABLE, CREATE PROCEDURE, CREATE FUNCTION, CREATE PACKAGE statements
- **Existing inventory** (optional, auto-detected) — `modernization-output/db-{slug}/oracle-inventory.json` from previous runs
- **Code pipeline artifacts** (optional, if available) — `modernization-output/{slug}/source-extracts.md` for app-level query patterns

## In-Scope & Out-of-Scope Objects

| Object Type | Provide DDL | Captured | Details |
|------------|------------|----------|---------|
| **Tables** | ✅ Yes | Full DDL: columns, types, precision, nullable, defaults, comments | Required |
| **Constraints** | ✅ Yes | PK, FK (with ON DELETE/UPDATE), unique, check, NOT NULL | In CREATE/ALTER TABLE |
| **Stored Procedures** | ✅ Yes | Full PLSQL body, parameters, variables, SQL, transaction control | Required |
| **Functions** | ✅ Yes | Full PLSQL body, parameters, return type, SQL | Required |
| **Packages** | ✅ Yes | Specification + body, all procedures/functions, package-level vars | Required |
| Triggers | ❌ No | Not expected (flag if referenced) | Out of scope |
| Sequences | ❌ No | Not expected (flag if referenced in DDL/PLSQL) | Out of scope |
| Views | ❌ No | Not expected (flag if referenced by PLSQL) | Out of scope |
| Jobs | ❌ No | Not expected | Out of scope |
| Synonyms / DB Links | ❌ No | Not expected (flag if referenced) | Out of scope |
| User-Defined Types | ❌ No | Not expected (flag if referenced as column/param type) | Out of scope |

For out-of-scope objects that are **referenced** by in-scope objects, the skill flags them as
dependency gaps so the user knows what else may need attention.

## Analysis Logic & Processing Steps

### Phase 1 — Parse & Catalog Provided DDL

For each CREATE TABLE provided, extract and catalog:

- **Columns**: name, data type (with precision, scale), nullable, default value, column comments
- **Primary Keys**: single and composite keys with constraint name
- **Foreign Keys**: referenced table/column, ON DELETE/UPDATE actions, constraint name
- **Unique Constraints**: constraint name, columns, deferrable/initially deferred flags
- **Check Constraints**: full constraint expression
- **NOT NULL Constraints**: per-column
- **Indexes**: inline (USING INDEX in constraint) or via CREATE INDEX statements
- **Partitioning**: if present — range/list/hash strategy, partition key columns
- **LOB Columns**: CLOB, BLOB — flag for Azure Blob Storage consideration
- **Table Comments**: any inline documentation

#### 1.2 Constraints

Consolidate all constraints from provided DDL:

- Every constraint across all tables: name, type, columns, expression, deferrable flags
- FK relationship graph: parent → child with cardinality indicators
- **Flag any FK that references a table NOT yet provided**: "⚠ FK references TABLE_X but DDL not yet provided"

#### 1.3 Stored Procedures & Functions

For each provided procedure/function, extract and preserve:

- **Full PLSQL body**: preserved verbatim, not summarized
- **Parameters**: name, type, direction (IN/OUT/INOUT), default values
- **Local variables & cursors**: all declarations with types
- **SQL statements**: every SELECT, INSERT, UPDATE, DELETE, MERGE — extract each one
- **Tables touched**: read set (SELECTs) and write set (INSERT/UPDATE/DELETE/MERGE)
- **Exception handling**: WHEN clauses and their actions
- **Transaction control**: COMMIT, ROLLBACK, SAVEPOINT, autonomous transactions
- **Procedure/function calls**: list all calls to other procedures/functions (flag if not provided yet)
- **Dynamic SQL**: capture EXECUTE IMMEDIATE patterns and constructed SQL signatures
- **Bulk operations**: FORALL, BULK COLLECT — note batch processing capability
- **Cursor loops**: cursor definitions and iteration patterns

#### 1.4 Packages

For each provided package, extract:

- **Package specification**: public interface — all procedure/function signatures, types, constants, exceptions, cursors
- **Package body**: full implementation of each procedure/function (same detail as Section 1.3)
- **Package-level variables**: state that persists across calls within a session
- **Initialization block**: package initialization logic
- **Inter-procedure dependencies**: which procedures call which within the package

### Phase 2 — Dependency Gap Detection

#### 2.1 Flag Missing Dependencies

Scan all provided DDL for references to objects NOT yet provided:

- **FK references to missing tables**: "⚠ DEPENDENCY GAP: TABLE_X (referenced by FK in TABLE_Y)"
- **PLSQL calls to missing procedures**: "⚠ DEPENDENCY GAP: PROC_Y (called by PROC_X, not yet provided)"
- **Sequence references**: "⚠ OUT OF SCOPE: SEQUENCE_Z (used in TABLE_X.COL default or PLSQL)"
- **Trigger references**: "⚠ OUT OF SCOPE: TRIGGER_T (referenced in DDL for TABLE_X)"
- **View references**: "⚠ OUT OF SCOPE: VIEW_V (referenced in PLSQL PROC_Y)"
- **Synonym/DB link references**: "⚠ OUT OF SCOPE: cross-schema dependency (synonym NAME or DB link)"
- **User-defined type references**: "⚠ OUT OF SCOPE: TYPE_T (used as column type or parameter type)"

Each gap includes context: which object references it and in what way.

#### 2.2 Incremental Merge

If `oracle-inventory.json` already exists from a prior run:

1. Read existing inventory
2. Add new tables/procedures/packages from current input
3. If a table is re-provided, overwrite the previous version (user is correcting)
4. Re-evaluate dependency gaps (a previously missing table may now be provided)
5. Update run_history with timestamp and what was added this run

### Phase 3 — Relationship Analysis

#### 3.1 Build Entity Relationship Map

From provided DDL, map all relationships:

- **Direct FK relationships**: parent → child with cardinality (1:1, 1:N, N:M)
- **Implicit relationships**: columns named `{TABLE}_ID` without FK constraints — flag as "possible relationship"
- **Join patterns from PLSQL**: tables frequently joined together in queries
- **Circular references**: tables that reference each other (directly or indirectly)
- **Self-referencing tables**: hierarchies or adjacency lists
- **Inheritance patterns**: tables with discriminator columns suggesting type hierarchies
- **Bridge/junction tables**: many-to-many relationship tables
- **For each relationship**: cascade behavior (ON DELETE CASCADE/SET NULL/RESTRICT), nullable FK columns

#### 3.2 Identify Data Access Patterns

From provided PLSQL code:

- Extract every query: SELECT statements, their WHERE clauses, JOIN patterns
- **Group by table**: which tables are queried most, write frequency
- **Identify common WHERE clause patterns** → partition key candidates for Cosmos DB (e.g., frequently filtered by CUSTOMER_ID)
- **Write patterns**: single-row inserts, bulk inserts (FORALL), update-heavy, append-only
- **Cross-reference with code artifacts** (if available): compare PLSQL patterns with app-level query patterns from source-extracts.md

### Phase 4 — Cosmos DB Readiness Assessment

#### 4.1 Flag Migration Challenges

For each provided object, assess and flag:

- **Wide tables** (50+ columns) → likely needs splitting into multiple document types
- **Tables with many FK references** → needs denormalization strategy (which FKs to embed, which to reference)
- **N:M relationships** → needs embedding decision (embed collection of IDs or link tables)
- **PLSQL with complex transaction logic** (multi-table COMMIT/ROLLBACK) → needs app-layer redesign
- **PLSQL with cursor loops** (row-by-row processing) → needs batch/bulk pattern redesign
- **LOB columns** (CLOB/BLOB) → needs Azure Blob Storage integration
- **Sequence references** → needs ID generation strategy (GUID, ULID, snowflake, etc.)
- **Cross-table constraints** → cross-document constraints need app-layer enforcement

#### 4.2 Self-Validation Pass

Confirm completeness:

- Every column in every provided table is captured
- Every parameter in every provided procedure is captured
- All FK relationships are mapped (even to missing tables)
- All dependency gaps are documented
- Report: "{N} tables, {M} procedures, {K} packages cataloged. {G} dependency gaps remaining."

## Output Specification

### Output 1 — `schema-analysis.md`

Comprehensive schema analysis in markdown format.

```markdown
# Oracle Schema Analysis: {schema-name}

## Schema Overview
- **Schema**: {name}
- **Total Tables Provided**: {count}
- **Total Stored Procedures Provided**: {count}
- **Total Functions Provided**: {count}
- **Total Packages Provided**: {count} ({total procedures across all packages})
- **Dependency Gaps**: {count}
- **Out-of-Scope References**: {count}
- **Run Mode**: {Initial | Incremental — added {N} tables, {M} procedures this run}

## ⚠ Dependency Gaps (Provide DDL for These Next)

| # | Object | Type | Referenced By | Context |
|---|--------|------|--------------|---------|
| 1 | CUSTOMERS | Table | ORDERS.CUSTOMER_ID (FK) | Foreign key reference |
| 2 | PROC_VALIDATE_ORDER | Procedure | PKG_ORDER_MGMT | Called but not provided |
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
**Calls**: {other procedures/functions called, flagging missing}
**Transaction Control**: {COMMIT, ROLLBACK, autonomous transaction flags}
**Cosmos DB Challenge**: {bulk loop processing, multi-table transactions, sequence dependencies}

**Full Body**:
```plsql
[Full PLSQL code preserved verbatim]
```

(Repeat for every provided procedure/function)

## Packages

### {PACKAGE_NAME}
**Public Interface**: {list of public procedures/functions with signatures}
**Package Variables**: {state that persists across calls}
**Procedures Defined**: {count}

#### Procedure: {PROC_NAME}
[Same format as Stored Procedures section above]

(Repeat for every procedure/function in the package)

## Relationship Analysis

### Foreign Key Graph

```
CUSTOMERS (1) ──1:N──> ORDERS
ORDERS (1) ──1:N──> ORDER_ITEMS
PRODUCTS (1) ──1:N──> ORDER_ITEMS
...
```

### Data Access Patterns

| Table | Read Frequency | Write Pattern | Common WHERE Clauses | Cosmos DB Partition Key Candidate |
|-------|----------------|----------------|---------------------|----------------------------------|
| ORDERS | High | Insert bulk, update status | CUSTOMER_ID, STATUS | CUSTOMER_ID |
| ORDER_ITEMS | High | Insert bulk | ORDER_ID, PRODUCT_ID | ORDER_ID |
| ... | ... | ... | ... | ... |

## Migration Complexity Assessment

### Overall Readiness: {Green | Yellow | Red}

**Summary**:
- {count} tables cataloged
- {count} straightforward tables (simple structure, clear partition strategy)
- {count} complex tables (wide, many relationships, denormalization needed)
- {count} PLSQL procedures with migration challenges (cursor loops, multi-table transactions)

**Key Recommendations**:
1. [Specific to this schema]
2. [Specific to this schema]
3. [Specific to this schema]
```

### Output 2 — `oracle-inventory.json`

Structured JSON inventory with complete object metadata and run history.

```json
{
  "run_history": [
    {
      "timestamp": "2026-03-22T15:30:00Z",
      "objects_added": {
        "tables": ["ORDERS", "CUSTOMERS"],
        "procedures": [],
        "functions": [],
        "packages": []
      },
      "mode": "Initial"
    },
    {
      "timestamp": "2026-03-22T16:00:00Z",
      "objects_added": {
        "tables": ["ORDER_ITEMS"],
        "procedures": ["PROC_CREATE_ORDER"],
        "functions": [],
        "packages": []
      },
      "mode": "Incremental"
    }
  ],
  "schema": "APP",
  "tables": [
    {
      "name": "ORDERS",
      "owner": "APP",
      "columns": [
        {
          "name": "ORDER_ID",
          "type": "NUMBER",
          "precision": 12,
          "scale": 0,
          "nullable": false,
          "default": null,
          "constraints": ["PK"],
          "comment": "Unique order identifier"
        },
        {
          "name": "CUSTOMER_ID",
          "type": "NUMBER",
          "precision": 12,
          "scale": 0,
          "nullable": false,
          "default": null,
          "constraints": ["FK→CUSTOMERS.ID"],
          "comment": "Reference to customer"
        }
      ],
      "constraints": [
        {
          "name": "PK_ORDERS",
          "type": "PRIMARY KEY",
          "columns": ["ORDER_ID"]
        },
        {
          "name": "FK_ORDERS_CUSTOMER",
          "type": "FOREIGN KEY",
          "columns": ["CUSTOMER_ID"],
          "references": {
            "table": "CUSTOMERS",
            "column": "ID"
          },
          "on_delete": "CASCADE",
          "on_update": null
        }
      ],
      "indexes": [
        {
          "name": "IDX_ORDERS_CUSTOMER",
          "columns": ["CUSTOMER_ID"],
          "unique": false,
          "type": "BTREE"
        }
      ],
      "partitioning": null,
      "referenced_by": ["ORDER_ITEMS"],
      "references": ["CUSTOMERS"],
      "cosmos_readiness": {
        "partition_key_candidate": "CUSTOMER_ID",
        "challenges": ["FK to CUSTOMERS", "1:N relationship to ORDER_ITEMS"]
      }
    }
  ],
  "procedures": [
    {
      "name": "PROC_CREATE_ORDER",
      "owner": "APP",
      "parameters": [
        {
          "name": "p_customer_id",
          "type": "NUMBER",
          "direction": "IN"
        },
        {
          "name": "p_order_id",
          "type": "NUMBER",
          "direction": "OUT"
        }
      ],
      "local_variables": [
        {
          "name": "v_count",
          "type": "INTEGER"
        }
      ],
      "tables_read": ["CUSTOMERS", "ORDERS"],
      "tables_written": ["ORDERS"],
      "procedure_calls": [],
      "sequence_references": ["ORDER_SEQ"],
      "transaction_control": ["COMMIT", "ROLLBACK"],
      "bulk_operations": false,
      "cursor_loops": false,
      "dynamic_sql": false,
      "plsql_body": "[Full procedure code preserved]",
      "cosmos_challenges": ["COMMIT/ROLLBACK on ORDERS", "FK validation on CUSTOMERS"]
    }
  ],
  "packages": [
    {
      "name": "PKG_ORDER_MGMT",
      "owner": "APP",
      "specification": "[Full package spec]",
      "procedures": [
        {
          "name": "CREATE_ORDER",
          "signature": "(p_customer_id IN NUMBER, p_order_id OUT NUMBER)",
          "body": "[Full procedure implementation]",
          "tables_read": ["CUSTOMERS"],
          "tables_written": ["ORDERS"]
        }
      ],
      "package_variables": [],
      "initialization_block": null
    }
  ],
  "dependency_gaps": [
    {
      "name": "CUSTOMERS",
      "type": "Table",
      "referenced_by": "ORDERS.FK_ORDERS_CUSTOMER",
      "context": "Foreign key reference — ORDERS.CUSTOMER_ID references CUSTOMERS.ID"
    },
    {
      "name": "PROC_VALIDATE_ORDER",
      "type": "Procedure",
      "referenced_by": "PKG_ORDER_MGMT.CREATE_ORDER",
      "context": "Called but not yet provided"
    }
  ],
  "out_of_scope_references": [
    {
      "name": "ORDER_SEQ",
      "type": "Sequence",
      "referenced_by": "ORDERS table (DEFAULT for ORDER_ID)",
      "context": "ID generation — user will need to implement GUID or ULID strategy"
    },
    {
      "name": "TRG_ORDER_AUDIT",
      "type": "Trigger",
      "referenced_by": "ORDERS table",
      "context": "Business logic — needs mapping to Cosmos DB change feed"
    }
  ]
}
```

### Output 3 — `relationship-map.md`

Entity relationship diagram and aggregate boundary analysis.

```markdown
# Entity Relationship Map: {schema-name}

## ER Diagram (Mermaid)

```mermaid
erDiagram
  CUSTOMERS ||--o{ ORDERS : places
  ORDERS ||--o{ ORDER_ITEMS : contains
  PRODUCTS ||--o{ ORDER_ITEMS : "ordered in"
  ...
```

## Relationship Summary

| From | To | Cardinality | FK Constraint | Cascade Behavior | Nullable | Notes |
|------|-----|-------------|--------------|-----------------|----------|-------|
| CUSTOMERS | ORDERS | 1:N | FK_ORDERS_CUSTOMER | CASCADE | No | Every order must have a customer |
| ORDERS | ORDER_ITEMS | 1:N | FK_ITEMS_ORDER | CASCADE | No | Order must have items |
| ... | ... | ... | ... | ... | ... | ... |

## Implicit Relationships (Not Explicitly Constrained)

| From | To | Pattern | Found In | Confidence |
|------|-----|---------|----------|------------|
| ORDERS | PRODUCTS | PRODUCT_ID column + join in PROC_CREATE_ORDER | PLSQL procedure | High |
| ... | ... | ... | ... | ... |

## Circular References & Hierarchies

- **DEPARTMENTS**: self-referencing (PARENT_DEPT_ID) — organizational hierarchy
- (List any circular references)

## Cosmos DB Aggregate Boundaries

Suggested document structures based on relationships:

### Aggregate 1: Order with Items
Embed ORDER_ITEMS directly in ORDERS document (1:N with cascade):
```json
{
  "order_id": 123,
  "customer_id": 45,
  "items": [
    { "product_id": 678, "qty": 2 },
    { "product_id": 679, "qty": 1 }
  ]
}
```

### Aggregate 2: Customer Profile
Keep customer separate (referenced), denormalize minimal fields in ORDERS if needed.

(Continue for other aggregates)
```

## Execution Steps

1. **Read existing inventory** (if present) — `oracle-inventory.json` in the output directory
2. **Parse user-provided DDL** — validate syntax, extract objects
3. **Catalog objects** — tables, constraints, procedures, packages (Phases 1-2)
4. **Detect gaps** — flag missing dependencies, out-of-scope references
5. **Merge incrementally** — add new objects, update existing, re-evaluate gaps
6. **Analyze relationships** — FK graph, implicit relationships, join patterns
7. **Assess migration readiness** — identify Cosmos DB challenges per object
8. **Self-validate** — confirm all objects captured, gaps documented
9. **Generate outputs** — schema-analysis.md, oracle-inventory.json, relationship-map.md

## Output Location

All outputs are written to:
```
modernization-output/db-{db-slug}/
  schema-analysis.md
  oracle-inventory.json
  relationship-map.md
```

Example: `/analyze-schema orders` → `modernization-output/db-orders/`

## Session & State Management

- **Incremental runs**: each invocation reads the existing inventory file and merges new objects
- **User corrections**: if a table is re-provided, the new version completely replaces the old
- **Timestamp tracking**: run_history records when each object was added
- **Persistent dependency gaps**: gaps carry forward until the user provides the missing DDL
