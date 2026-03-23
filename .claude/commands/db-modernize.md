# Database Modernize — Oracle → Cosmos DB Pipeline

Run the database modernization pipeline for an Oracle schema: design document model → generate migration artifacts → validate migration.

**Pre-requisite**: `/analyze-schema` must have been run (possibly multiple times) to build the inventory. This command picks up from a completed analysis.

## Arguments
- `$ARGUMENTS` — expects: `<db-slug>`
  - **db-slug**: The schema/module name (e.g., `orders`, `payments`). Must match an existing `modernization-output/db-{slug}/` directory.

## Execution Flow

### Step 1: Verify Analysis is Complete
1. Check that `modernization-output/db-{slug}/` exists
2. Check that `oracle-inventory.json`, `schema-analysis.md`, and `relationship-map.md` exist
3. Read `oracle-inventory.json` and check `dependency_gaps`:
   - If **no gaps**: proceed normally
   - If **gaps remain**: warn the user with the list of missing objects. Ask: "There are {N} dependency gaps. Continue anyway, or run `/analyze-schema {db-slug}` to provide the missing DDL first?"
4. Check for code pipeline artifacts in related slugs (`modernization-output/{slug}/source-extracts.md`, `techspec.md`) — these provide optional context for better access pattern analysis

### Step 2: Design Document Model
Run the **design-documents** skill:
- Input: `db-{slug}/schema-analysis.md` + `oracle-inventory.json` + `relationship-map.md`
- Optional: code pipeline artifacts for access pattern context
- Outputs saved to `db-{slug}/`:
  - `document-model.md` — containers, partition keys, document schemas, denormalization
  - `cosmos-config.json` — Cosmos DB configuration (containers, indexing, throughput)
  - `access-patterns.md` — access pattern catalog

**Validation gate**: Confirm all 3 files were created. Confirm at least one container is defined in cosmos-config.json.

### Step 3: Generate Migration Artifacts
Run the **gen-migration** skill:
- Input: All `db-{slug}/` artifacts
- Outputs saved to `db-{slug}/migration/`:
  - `infrastructure/` — Bicep/Terraform provisioning, Azure Functions project
  - `transform-scripts/` — per-table Python transformation scripts
  - `export-queries/` — Oracle SELECT statements for extraction
  - `adf-pipeline.json` — Azure Data Factory pipeline
  - `validation/` — pre/post migration checks
  - `seed-data/` — reference/static data
  - `rollback-plan.md` — rollback and cutover strategy

**Validation gate**: Confirm migration/ directory was created with at least transform-scripts and rollback-plan.md.

### Step 4: Validate Migration Design
Run the **validate-migration** skill:
- Input: All `db-{slug}/` artifacts including migration/
- Output: `db-{slug}/validation-report.md`

### Step 5: Report Results

```
## Database Modernization Complete: db-{slug}

### Artifacts Generated
| Step | Artifact | Path | Status |
|------|----------|------|--------|
| 0 | Schema Analysis | db-{slug}/schema-analysis.md | ✅ (from prior /analyze-schema runs) |
| 0 | Oracle Inventory | db-{slug}/oracle-inventory.json | ✅ |
| 0 | Relationship Map | db-{slug}/relationship-map.md | ✅ |
| 1 | Document Model | db-{slug}/document-model.md | ✅ |
| 1 | Cosmos Config | db-{slug}/cosmos-config.json | ✅ |
| 1 | Access Patterns | db-{slug}/access-patterns.md | ✅ |
| 2 | Migration Artifacts | db-{slug}/migration/ | ✅ |
| 3 | Validation Report | db-{slug}/validation-report.md | ✅ |

### Inventory Summary
- Tables analyzed: {count}
- Procedures/Functions analyzed: {count}
- Packages analyzed: {count}
- Dependency gaps: {count} {⚠ if > 0}
- Out-of-scope references: {count}

### Cosmos DB Design Summary
- Containers: {count}
- Document types: {count}
- Partition key strategies: {list}

### Validation Summary
{Extract overall status and key findings from validation-report.md}

### Next Steps
1. Review document-model.md — verify partition key choices and denormalization decisions
2. Review migration/rollback-plan.md — confirm cutover strategy
3. If dependency gaps exist, provide remaining DDL via /analyze-schema and re-run
4. If code pipeline exists, re-run /gen-code to generate Spring Data Cosmos DB repositories
5. Execute migration in a test environment first
```

## Error Handling
- If analysis artifacts don't exist → stop with: "Run /analyze-schema {db-slug} first to build the inventory"
- If dependency gaps exist → warn but allow the user to proceed or go back
- If any step fails → report which step failed, do NOT skip steps
- If cosmos-config.json would have zero containers → stop and report the issue
