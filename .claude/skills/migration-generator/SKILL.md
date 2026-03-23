---
name: gen-migration
description: Generate database migration artifacts including infrastructure provisioning, data transformation scripts, ETL pipelines, validation queries, and rollback plans. Trigger on migration scripts, data migration, ETL, Azure Data Factory, transform, Oracle export, Cosmos DB import, migration pipeline, rollback plan.
---

# Migration Generator (gen-migration)

## Overview

This skill generates the complete, executable migration artifacts that transform data from a legacy Oracle database to Azure Cosmos DB. The generated outputs translate the document model design (from `gen-document-model`) into concrete migration code: infrastructure-as-code provisioning scripts, per-table data transformation scripts, Azure Data Factory pipeline definitions, comprehensive validation queries, seed data, and a detailed rollback plan.

The primary goal is to produce a **repeatable, testable, production-ready migration** that maintains data integrity, preserves legacy IDs for traceability, and enables zero-downtime cutover via blue-green deployment and feature flags.

## Inputs

You will receive a **slug** (the modernization-output directory). Load these files from `modernization-output/{slug}/`:

- **schema-analysis.md** — detailed inventory of Oracle tables, columns, and constraints provided by the user; estimated row counts
- **oracle-inventory.json** — machine-readable inventory with:
  - `tables`: full DDL catalog (columns, constraints)
  - `procedures`, `packages`, `functions`: PLSQL bodies
  - `dependency_gaps`: tables/procedures referenced but not yet provided (gaps for user to resolve)
  - `out_of_scope_refs`: references to triggers, sequences, views, jobs, synonyms, types (not provided as full objects, only flagged for awareness)
- **document-model.md** — Cosmos DB container designs, partition keys, embedded relationships, denormalization decisions, change feed processor mappings
- **cosmos-config.json** — Cosmos DB configuration (container names, partition keys, unique keys, indexing policies, throughput, change feed policies)
- **access-patterns.md** — catalog of query patterns and their implementation across Oracle and Cosmos DB

## Handling Out-of-Scope References

Before generating migration artifacts, **scan oracle-inventory.json for out_of_scope_refs and dependency_gaps**:

### Sequences (`out_of_scope_refs`)
- Oracle sequences provide auto-increment IDs; Cosmos DB has no equivalent
- **Action**: Default to GUID generation (per `id_generator.py`). Document in migration notes: "ID generation strategy: GUID instead of Oracle sequence {SEQUENCE_NAME}"
- If legacy application code depends on sequential IDs for ordering, note alternative: "Sort by `_metadata.createdAt` instead of sequence order"

### Triggers (`out_of_scope_refs`)
- Oracle trigger bodies are not provided in the inventory (only the fact they exist is flagged)
- **Action**: For each trigger referenced in document-model.md, the change feed processor mapping tells you what to implement. Note: "Trigger body not available — confirm business logic with team before deploying change feed processor"
- Generate comments in Azure Function code: `// Oracle trigger {TRIGGER_NAME} logic — VERIFY WITH TEAM`

### Views, Jobs, Synonyms, Types (`out_of_scope_refs`)
- Not provided as full definitions, only flagged as referenced
- **Action**: Flag in generation report with recommendation: "⚠ {OBJECT_TYPE} {NAME} referenced — plan separate handling or refactor queries"
- Do NOT attempt to transform these; they require manual review

### Dependency Gaps (`dependency_gaps`)
- Tables or procedures referenced but not yet provided by user
- **Action**: Warn and continue if gaps exist (migration can proceed for provided objects). Generate report line: "⚠ Dependency Gap: {OBJECT_NAME} (type: {type}) referenced by {REFERENCED_BY} — provide DDL if needed"

## Target Architecture

Migration targets **Azure Cosmos DB** as the destination, with supporting infrastructure:

### Infrastructure
- **Cosmos DB**: Serverless or autoscale containers with change feed enabled
- **Azure Data Factory**: Orchestrates the extraction, transformation, and load (ETL)
- **Azure Functions**: Processes change feed events (replaces Oracle triggers)
- **Azure Key Vault**: Stores connection strings and credentials
- **Azure Blob Storage**: Stages CSV/Parquet files and large objects (LOBs)
- **Azure Storage Queue**: Dead-letter pattern for failed documents

### Data Transformation
- **Python scripts** (preferred) or Node.js for row-by-row transformation
- **Type converters**: Oracle types → JSON-compatible types (DATE → ISO 8601, CLOB → string, RAW → base64)
- **ID generation**: New IDs (GUID/ULID) with legacy ID preservation (`legacyId` field)
- **Denormalization**: Embed child records into parent documents per document-model.md
- **Metadata injection**: `_metadata` object with `createdAt`, `migratedAt`, `version`, `source`

## Generation Phases

Execute migration code generation in **5 distinct phases**:

### Phase 1 — Infrastructure Setup

Generate infrastructure-as-code for Cosmos DB provisioning:

1. **Terraform Module** (`infrastructure/terraform/main.tf`):
   ```hcl
   terraform {
     required_providers {
       azurerm = {
         source  = "hashicorp/azurerm"
         version = "~> 3.0"
       }
     }
   }

   provider "azurerm" {
     features {}
   }

   resource "azurerm_resource_group" "migration" {
     name     = "rg-${var.slug}-migration"
     location = var.azure_region
   }

   resource "azurerm_cosmosdb_account" "cosmos" {
     name                = "cosmos-${var.slug}-${random_string.cosmos_suffix.result}"
     location            = azurerm_resource_group.migration.location
     resource_group_name = azurerm_resource_group.migration.name
     offer_type          = "Standard"

     consistency_policy {
       consistency_level       = "BoundedStaleness"
       max_interval_in_seconds = 86400
       max_staleness_prefix    = 100000
     }

     geo_location {
       location          = azurerm_resource_group.migration.location
       failover_priority = 0
     }
   }

   resource "azurerm_cosmosdb_sql_database" "database" {
     name                = var.cosmos_database_name
     resource_group_name = azurerm_resource_group.migration.name
     account_name        = azurerm_cosmosdb_account.cosmos.name
   }

   resource "azurerm_cosmosdb_sql_container" "containers" {
     for_each            = var.containers
     name                = each.value.name
     resource_group_name = azurerm_resource_group.migration.name
     account_name        = azurerm_cosmosdb_account.cosmos.name
     database_name       = azurerm_cosmosdb_sql_database.database.name
     partition_key_path  = each.value.partition_key

     throughput = lookup(each.value, "throughput_type", "") == "autoscale" ? null : each.value.throughput

     dynamic "autoscale_settings" {
       for_each = lookup(each.value, "throughput_type", "") == "autoscale" ? [each.value.autoscale_settings] : []
       content {
         max_throughput = autoscale_settings.value.max_ru
       }
     }

     unique_key {
       paths = each.value.unique_keys
     }

     indexing_policy {
       indexing_mode = "consistent"
       included_path {
         path = "/*"
       }
       excluded_path {
         path = "/_etag/?"
       }
     }
   }

   resource "azurerm_cosmosdb_sql_container" "leases" {
     name                = "leases"
     resource_group_name = azurerm_resource_group.migration.name
     account_name        = azurerm_cosmosdb_account.cosmos.name
     database_name       = azurerm_cosmosdb_sql_database.database.name
     partition_key_path  = "/id"
     throughput          = 400
   }

   output "cosmos_endpoint" {
     value = azurerm_cosmosdb_account.cosmos.endpoint
   }

   output "cosmos_key" {
     value     = azurerm_cosmosdb_account.cosmos.primary_key
     sensitive = true
   }
   ```

2. **Terraform variables** (`infrastructure/terraform/variables.tf`):
   ```hcl
   variable "slug" {
     type        = string
     description = "Modernization slug"
   }

   variable "azure_region" {
     type        = string
     default     = "eastus"
   }

   variable "cosmos_database_name" {
     type        = string
   }

   variable "containers" {
     type = map(object({
       name           = string
       partition_key  = string
       throughput     = optional(number, 400)
       throughput_type = optional(string, "provisioned")
       autoscale_settings = optional(object({
         max_ru = number
       }))
       unique_keys = optional(list(string), [])
     }))
   }
   ```

3. **Terraform outputs** (`infrastructure/terraform/outputs.tf`) — capture connection strings and container names for downstream scripts.

4. **Azure Functions project** (`infrastructure/azure-functions/`):
   - Generate `host.json`, `local.settings.json` (template with placeholders)
   - Create one function per change feed processor mapping from document-model.md (which maps Oracle triggers to change feed handlers)
   - Each function receives Cosmos DB trigger, logs changes, applies business logic (e.g., audit trail, aggregation)
   - **Important**: Oracle trigger bodies are not available (flagged as `out_of_scope_refs`). Generate function templates with TODO comments for business logic: `// TODO: confirm {ORACLE_TRIGGER_NAME} behavior with team`. Add note in rollback-plan.md: "Change feed processors implement Oracle trigger logic — confirm with team before production"

5. **Bicep Alternative** (`infrastructure/bicep/main.bicep`):
   - Generate equivalent Bicep template for teams preferring Bicep over Terraform
   - Include parameter definitions, resource declarations, outputs

### Phase 2 — Data Transformation Scripts

For each Oracle table → Cosmos DB container mapping from document-model.md:

1. **Per-table transformation script** (Python, `transform-scripts/{table_name}.py`):
   ```python
   import json
   import csv
   import uuid
   from datetime import datetime
   from typing import Any, Dict, List
   import logging

   logging.basicConfig(level=logging.INFO)
   logger = logging.getLogger(__name__)

   class OracleToCosmosTransformer:
       """
       Transform Oracle {TABLE_NAME} rows to Cosmos DB {CONTAINER_NAME} documents.

       Legacy Source: Oracle {TABLE_NAME} (analyzed {ANALYSIS_DATE})
       Target Container: {CONTAINER_NAME}
       Partition Key: {PARTITION_KEY}
       """

       def __init__(self, input_csv: str, output_json: str):
           self.input_csv = input_csv
           self.output_json = output_json
           self.processed_count = 0
           self.error_count = 0
           self.legacy_ids = {}

       def transform_row(self, row: Dict[str, Any]) -> Dict[str, Any]:
           """
           Transform a single Oracle row to a Cosmos DB document.

           Applies:
           - Type conversions (Oracle → JSON)
           - ID generation (new GUID + legacy ID preservation)
           - Metadata injection
           - Denormalization (child records embedded)
           - Null handling
           """
           try:
               # Preserve legacy ID for traceability
               legacy_id = row.get('{LEGACY_ID_COLUMN}')

               # Generate new ID (GUID/ULID)
               new_id = str(uuid.uuid4())
               self.legacy_ids[legacy_id] = new_id

               # Base document
               doc = {
                   "id": new_id,
                   "type": "{DOCUMENT_TYPE}",
                   "_metadata": {
                       "createdAt": datetime.utcnow().isoformat() + "Z",
                       "migratedAt": datetime.utcnow().isoformat() + "Z",
                       "version": "1.0",
                       "source": "oracle",
                       "legacyId": legacy_id,
                       "legacyTable": "{TABLE_NAME}"
                   }
               }

               # Type conversions
               # Date/Timestamp → ISO 8601
               if row.get('{DATE_COLUMN}'):
                   doc['{date_field}'] = self._convert_date(row['{DATE_COLUMN}'])

               # Number → int or float
               if row.get('{NUMBER_COLUMN}'):
                   doc['{number_field}'] = self._convert_number(row['{NUMBER_COLUMN}'])

               # CLOB → string (truncate if > 1MB)
               if row.get('{CLOB_COLUMN}'):
                   doc['{text_field}'] = self._convert_clob(row['{CLOB_COLUMN}'])

               # RAW → base64
               if row.get('{RAW_COLUMN}'):
                   doc['{binary_field}'] = self._convert_raw(row['{RAW_COLUMN}'])

               # Partition key (must be populated)
               partition_value = row.get('{PARTITION_KEY_COLUMN}')
               if not partition_value:
                   raise ValueError(f"Partition key '{PARTITION_KEY_COLUMN}' missing for legacy ID {legacy_id}")
               doc['{partition_key_field}'] = partition_value

               # Embed child records (denormalization per document-model.md)
               # Example: Order with embedded OrderItems
               if "{CHILD_TABLE}" in row:
                   doc['items'] = self._embed_children(row["{CHILD_TABLE}"])

               self.processed_count += 1
               return doc

           except Exception as e:
               logger.error(f"Error transforming row {legacy_id}: {str(e)}")
               self.error_count += 1
               raise

       def _convert_date(self, oracle_date: str) -> str:
           """Convert Oracle DATE/TIMESTAMP to ISO 8601."""
           if not oracle_date:
               return None
           # Assumes CSV export format like '2023-01-15 14:30:00'
           try:
               dt = datetime.strptime(oracle_date, "%Y-%m-%d %H:%M:%S")
               return dt.isoformat() + "Z"
           except ValueError:
               return oracle_date  # Pass through if already ISO

       def _convert_number(self, oracle_number: str) -> float:
           """Convert Oracle NUMBER to float or int."""
           if not oracle_number or oracle_number.lower() == 'null':
               return None
           try:
               val = float(oracle_number)
               return int(val) if val.is_integer() else val
           except ValueError:
               logger.warning(f"Failed to convert number: {oracle_number}")
               return None

       def _convert_clob(self, clob_value: str) -> str:
           """Convert Oracle CLOB, truncating if > 1MB."""
           if not clob_value:
               return None
           max_size = 1024 * 1024  # 1MB
           if len(clob_value) > max_size:
               logger.warning(f"CLOB truncated from {len(clob_value)} to {max_size} bytes")
               return clob_value[:max_size]
           return clob_value

       def _convert_raw(self, raw_value: str) -> str:
           """Convert Oracle RAW to base64."""
           if not raw_value:
               return None
           import base64
           try:
               return base64.b64encode(raw_value.encode()).decode('utf-8')
           except Exception as e:
               logger.error(f"Failed to encode RAW: {str(e)}")
               return None

       def _embed_children(self, children_data: List[Dict]) -> List[Dict]:
           """Embed child records into parent document."""
           embedded = []
           for child in children_data:
               embedded_child = {
                   "id": str(uuid.uuid4()),
                   "legacyId": child.get('{CHILD_ID_COLUMN}')
               }
               # Map child fields
               for field_name, value in child.items():
                   embedded_child[field_name] = value
               embedded.append(embedded_child)
           return embedded

       def process(self):
           """Read CSV, transform rows, write JSONL."""
           documents = []
           with open(self.input_csv, 'r', encoding='utf-8') as infile:
               reader = csv.DictReader(infile)
               for row_num, row in enumerate(reader, start=1):
                   try:
                       doc = self.transform_row(row)
                       documents.append(doc)
                   except Exception as e:
                       logger.error(f"Row {row_num} failed: {str(e)}")
                       self.error_count += 1
                       continue

           # Write JSONL (one JSON object per line for bulk import)
           with open(self.output_json, 'w', encoding='utf-8') as outfile:
               for doc in documents:
                   outfile.write(json.dumps(doc) + '\n')

           logger.info(f"Processed {self.processed_count} rows, {self.error_count} errors")
           logger.info(f"Output: {self.output_json}")

   if __name__ == "__main__":
       import sys
       input_csv = sys.argv[1]  # Oracle CSV export
       output_json = sys.argv[2]  # Cosmos DB JSONL

       transformer = OracleToCosmosTransformer(input_csv, output_json)
       transformer.process()
   ```

2. **Type converter helpers** (`transform-scripts/converters.py`):
   ```python
   """Oracle → Cosmos DB type converters."""

   def oracle_number_to_json(oracle_number: str) -> float:
       """Oracle NUMBER(p,s) → JSON number."""
       return float(oracle_number) if oracle_number else None

   def oracle_date_to_iso8601(oracle_date: str) -> str:
       """Oracle DATE → ISO 8601."""
       from datetime import datetime
       if not oracle_date:
           return None
       dt = datetime.strptime(oracle_date, "%Y-%m-%d %H:%M:%S")
       return dt.isoformat() + "Z"

   def oracle_clob_to_string(clob: str, max_size: int = 1048576) -> str:
       """Oracle CLOB → JSON string (truncate if > max_size)."""
       if not clob:
           return None
       if len(clob) > max_size:
           return clob[:max_size]
       return clob

   def oracle_raw_to_base64(raw: bytes) -> str:
       """Oracle RAW → base64."""
       import base64
       return base64.b64encode(raw).decode('utf-8') if raw else None

   def oracle_char_to_string(char: str) -> str:
       """Oracle CHAR/VARCHAR2 → JSON string."""
       return char.rstrip() if char else None
   ```

3. **ID generation strategy** (`transform-scripts/id_generator.py`):

   Note: Oracle sequences referenced in `out_of_scope_refs` are replaced by GUID generation below. If sequence-like behavior is needed (e.g., sortable IDs), use ULID instead.

   ```python
   """ID generation with legacy ID preservation.

   Cosmos DB has no auto-increment sequences. This module replaces Oracle sequences with:
   - GUID: default, globally unique, random
   - ULID: sortable, time-ordered, preserves insertion order

   Legacy IDs (from Oracle sequences) are preserved in _metadata.legacyId for traceability.
   """

   import uuid
   from datetime import datetime

   def generate_guid() -> str:
       """Generate new GUID for Cosmos DB id field."""
       return str(uuid.uuid4())

   def generate_ulid() -> str:
       """Generate new ULID (sortable, time-ordered).

       Use instead of GUID if application relies on ID ordering.
       ULID preserves insertion order and is sortable as a string.
       """
       from ulid import ULID
       return str(ULID())

   def preserve_legacy_id(legacy_id: str, new_id: str) -> dict:
       """Record mapping from legacy sequence-generated ID to new ID.

       Stored in _metadata for audit trail and potential reverse-lookups.
       Example: legacy_id = 1000001 (Oracle ORDERS.ORDER_ID generated by sequence)
               new_id = "c1234-5678..." (new GUID)
       """
       return {
           "legacyId": legacy_id,
           "newId": new_id,
           "migratedAt": datetime.utcnow().isoformat() + "Z"
       }
   ```

4. **Denormalization helpers** (`transform-scripts/denormalization.py`):
   ```python
   """Denormalization and embedding child records."""

   def embed_one_to_many(parent_doc: dict, child_records: list, child_key: str) -> dict:
       """Embed a one-to-many relationship into parent."""
       parent_doc[child_key] = child_records
       return parent_doc

   def materialize_denormalized_view(source_docs: list, view_key: str) -> dict:
       """Generate denormalized summary view from source documents."""
       # Example: VW_ORDER_SUMMARY → materialized document combining order + items + customer
       summary = {
           "id": str(uuid.uuid4()),
           "type": "summary",
           "orders": source_docs,
           "_metadata": {"view": view_key}
       }
       return summary
   ```

### Phase 3 — Azure Data Factory Pipeline

Generate ADF pipeline definition (`adf-pipeline.json`):

```json
{
  "name": "{slug}_oracle_to_cosmos_pipeline",
  "properties": {
    "description": "Migrate {schema_name} from Oracle to Cosmos DB. Generated {timestamp}.",
    "activities": [
      {
        "name": "PreMigrationValidation",
        "type": "ExecuteDataFlow",
        "dependsOn": [],
        "policy": {
          "timeout": "7.00:00:00",
          "retry": 0,
          "retryIntervalInSeconds": 30,
          "secureOutput": false,
          "secureInput": false
        },
        "typeProperties": {
          "dataflow": {
            "referenceName": "PreMigrationValidation",
            "type": "DataFlowReference"
          },
          "staging": {
            "linkedService": {
              "referenceName": "AzureBlobStorageLinkedService",
              "type": "LinkedServiceReference"
            },
            "path": "staging/pre-migration"
          },
          "compute": {
            "coreCount": 8,
            "computeType": "General"
          }
        }
      },
      {
        "name": "ExtractFromOracle",
        "type": "Copy",
        "dependsOn": [
          {
            "activity": "PreMigrationValidation",
            "dependencyConditions": ["Succeeded"]
          }
        ],
        "policy": {
          "timeout": "7.00:00:00",
          "retry": 2,
          "retryIntervalInSeconds": 30
        },
        "typeProperties": {
          "source": {
            "type": "OracleSource",
            "oracleReaderQuery": "@concat('SELECT * FROM {TABLE_NAME} ORDER BY {PRIMARY_KEY}')",
            "partitionOption": "DynamicRange",
            "partitionSettings": {
              "partitionColumnName": "{PRIMARY_KEY}",
              "partitionLowerBound": "@activity('GetMinId').output.firstRow.min_id",
              "partitionUpperBound": "@activity('GetMaxId').output.firstRow.max_id",
              "partitionSize": 10000
            }
          },
          "sink": {
            "type": "DelimitedTextSink",
            "storeSettings": {
              "type": "AzureBlobStorageWriteSettings"
            },
            "formatSettings": {
              "type": "DelimitedTextWriteSettings",
              "quoteAllText": true,
              "fileExtension": ".csv"
            }
          },
          "translator": {
            "type": "TabularTranslator",
            "mappings": [
              {
                "source": {"name": "{ORACLE_COLUMN}", "type": "String"},
                "sink": {"name": "{COLUMN_NAME}"}
              }
            ]
          }
        },
        "inputs": [
          {
            "referenceName": "OracleDataset",
            "type": "DatasetReference"
          }
        ],
        "outputs": [
          {
            "referenceName": "BlobCSVDataset",
            "type": "DatasetReference",
            "parameters": {
              "FileName": "{table_name}_extract.csv",
              "Path": "staging/extracts"
            }
          }
        ]
      },
      {
        "name": "TransformData",
        "type": "ExecuteDataFlow",
        "dependsOn": [
          {
            "activity": "ExtractFromOracle",
            "dependencyConditions": ["Succeeded"]
          }
        ],
        "typeProperties": {
          "dataflow": {
            "referenceName": "TransformOracleToCosmosDataFlow",
            "type": "DataFlowReference"
          },
          "staging": {
            "linkedService": {
              "referenceName": "AzureBlobStorageLinkedService",
              "type": "LinkedServiceReference"
            },
            "path": "staging/transforms"
          },
          "compute": {
            "coreCount": 16,
            "computeType": "General"
          }
        }
      },
      {
        "name": "LoadToCosmosDB",
        "type": "Copy",
        "dependsOn": [
          {
            "activity": "TransformData",
            "dependencyConditions": ["Succeeded"]
          }
        ],
        "policy": {
          "timeout": "7.00:00:00",
          "retry": 3,
          "retryIntervalInSeconds": 60
        },
        "typeProperties": {
          "source": {
            "type": "JsonSource",
            "storeSettings": {
              "type": "AzureBlobStorageReadSettings",
              "recursive": true,
              "wildcardFileName": "*.jsonl"
            }
          },
          "sink": {
            "type": "CosmosDbSqlApiSink",
            "writeBehavior": "upsert"
          },
          "enableStaging": true,
          "stagingSettings": {
            "linkedService": {
              "referenceName": "AzureBlobStorageLinkedService",
              "type": "LinkedServiceReference"
            },
            "path": "staging/cosmosdb-import"
          }
        },
        "inputs": [
          {
            "referenceName": "JsonlDataset",
            "type": "DatasetReference",
            "parameters": {
              "Path": "staging/transforms"
            }
          }
        ],
        "outputs": [
          {
            "referenceName": "CosmosDbDataset",
            "type": "DatasetReference",
            "parameters": {
              "ContainerName": "{CONTAINER_NAME}"
            }
          }
        ]
      },
      {
        "name": "PostMigrationValidation",
        "type": "ExecuteDataFlow",
        "dependsOn": [
          {
            "activity": "LoadToCosmosDB",
            "dependencyConditions": ["Succeeded"]
          }
        ],
        "typeProperties": {
          "dataflow": {
            "referenceName": "PostMigrationValidation",
            "type": "DataFlowReference"
          },
          "staging": {
            "linkedService": {
              "referenceName": "AzureBlobStorageLinkedService",
              "type": "LinkedServiceReference"
            },
            "path": "staging/post-migration"
          }
        }
      }
    ]
  }
}
```

### Phase 4 — Validation Queries

Generate validation scripts to verify migration integrity:

1. **Pre-migration validation** (`validation/pre-migration.sql`):
   ```sql
   -- Pre-migration data profiling (run against Oracle)
   -- Generated {timestamp}

   -- Row counts per table
   SELECT 'ORDERS' as table_name, COUNT(*) as row_count FROM ORDERS
   UNION ALL
   SELECT 'ORDER_ITEMS', COUNT(*) FROM ORDER_ITEMS
   UNION ALL
   SELECT 'CUSTOMERS', COUNT(*) FROM CUSTOMERS;

   -- Checksums for data integrity
   SELECT 'ORDERS', DBMS_CRYPTO.HASH(utl_raw.cast_to_raw(TO_CHAR(COUNT(*)) || MIN(ID) || MAX(ID)), 3)
   FROM ORDERS;

   -- Null value counts
   SELECT 'ORDERS', SUM(CASE WHEN ORDER_NUMBER IS NULL THEN 1 ELSE 0 END) as null_order_numbers
   FROM ORDERS;

   -- Sample records (first 10, last 10, random 10)
   SELECT * FROM ORDERS ORDER BY ID ASC FETCH FIRST 10 ROWS ONLY;
   ```

2. **Post-migration validation** (`validation/post-migration.sql`):
   ```sql
   -- Post-migration verification (run against Cosmos DB via SQL API)
   -- Generated {timestamp}

   -- Document counts per container
   SELECT c.type, COUNT(1) as document_count
   FROM c
   GROUP BY c.type;

   -- Verify all legacy IDs are preserved
   SELECT COUNT(1) as unmapped_legacy_ids
   FROM c
   WHERE c._metadata.legacyId IS NULL;

   -- Sample document verification
   SELECT TOP 10 c.id, c._metadata.legacyId, c._metadata.migratedAt
   FROM c
   ORDER BY c._metadata.migratedAt DESC;
   ```

3. **Reconciliation report** (`validation/reconciliation.py`):
   ```python
   import json
   from typing import Dict, List

   def reconcile_migration(oracle_stats: Dict, cosmos_stats: Dict) -> Dict:
       """Generate reconciliation report comparing Oracle and Cosmos DB."""
       report = {
           "timestamp": datetime.utcnow().isoformat(),
           "status": "PASS" if oracle_stats == cosmos_stats else "FAIL",
           "details": {
               "oracle": oracle_stats,
               "cosmos": cosmos_stats,
               "discrepancies": []
           }
       }

       for table, count in oracle_stats.items():
           cosmos_count = cosmos_stats.get(table, 0)
           if count != cosmos_count:
               report["details"]["discrepancies"].append({
                   "table": table,
                   "oracle_count": count,
                   "cosmos_count": cosmos_count,
                   "delta": cosmos_count - count
               })

       return report
   ```

### Phase 5 — Rollback Plan

Generate detailed rollback and cutover strategy:

1. **Rollback plan** (`rollback-plan.md`):
   ```markdown
   # Rollback Plan

   ## Scope
   This plan covers rollback of the {schema_name} migration from Oracle to Cosmos DB.
   Rollback can occur at any time before the feature flag switch to Cosmos DB is enabled.

   ## Strategy 1: Point-in-Time Restore (0-30 days post-migration)

   ### Prerequisites
   - Cosmos DB automated backup enabled (default: 30-day retention)
   - Oracle system remains online (parallel running during cutover window)

   ### Procedure
   1. Monitor Cosmos DB for data quality issues
   2. If issues detected:
      a. Stop writing to Cosmos DB (feature flag OFF)
      b. Continue serving reads from Oracle
      c. Restore Cosmos DB from backup to point-in-time via Azure Portal:
         - Navigate to Cosmos DB account → Backup & Restore
         - Select restore timestamp (pre-issue)
         - Click "Restore"
      d. Validate restored data against Oracle
      e. Resume migration if issues resolved

   ### RTO: 15-30 minutes
   ### RPO: Point-in-time (typically < 5 minutes)

   ## Strategy 2: Blue-Green Cutover

   ### Setup (pre-migration)
   - **Blue**: Oracle ORDERS table (production)
   - **Green**: Cosmos DB orders container (staging)
   - Feature flags control traffic routing

   ### Cutover Process
   1. Enable dual-write during load test phase:
      - New orders written to both Oracle and Cosmos DB
      - Reads: Blue (Oracle) until validation complete
   2. Validate Cosmos DB row-by-row against Oracle (reconciliation reports)
   3. Once 100% data parity achieved:
      - Enable feature flag: `migration.orders.use-cosmos=true`
      - Route new reads to Cosmos DB (Green)
      - Monitor error rates (should be 0)
   4. If issues detected:
      - Disable feature flag immediately
      - Route traffic back to Oracle (Blue)
      - Analyze discrepancies
      - Fix transformation logic and retry

   ### Rollback: Feature flag OFF immediately

   ## Strategy 3: Gradual Cutover (Canary)

   ### Phase A: Shadow Mode (Week 1-2)
   - All writes: Oracle only
   - Transformation pipeline runs in shadow mode
   - Data written to Cosmos DB (not served)
   - No customer impact if Cosmos DB fails
   - Action: Validate data parity daily

   ### Phase B: Canary (Week 3)
   - Route 5% of read traffic to Cosmos DB
   - Monitor error rates and latency
   - If error rate > 0.1%, rollback
   - If OK, increase to 25%

   ### Phase C: Ramp (Week 4)
   - Route 25%, then 50%, then 75%, then 100%
   - 1 week per ramp increment
   - Monitor SLA metrics continuously
   - Rollback threshold: any error rate increase > 1%

   ### Phase D: Blue Decommission (Week 5+)
   - Once 100% traffic on Cosmos DB for 2 weeks without issues
   - Decommission Oracle ORDERS table (after 90-day legal hold)

   ## Rollback Checklist

   - [ ] Feature flags accessible in code (`feature-toggle.yml`)
   - [ ] Monitoring dashboards display error rates per system (Oracle vs Cosmos)
   - [ ] Alerting configured: alert on error rate > 1%
   - [ ] Runbook documented and tested: "How to switch back to Oracle"
   - [ ] Database credentials stored in Key Vault (never in code)
   - [ ] Backup restored monthly and validated (DR test)

   ## Post-Migration Monitoring (30 days)

   Monitor these metrics:
   - Cosmos DB RU consumption vs forecast
   - Query latency (p50, p95, p99)
   - Error rates (network, validation, throttling)
   - Legacy ID lookup performance (for joins with old systems)
   - Change feed lag (if using change feed processors)

   If any metric exceeds SLA, trigger incident response:
   1. Revert feature flag (if applicable)
   2. Analyze root cause
   3. Fix and re-deploy
   4. Resume migration

   ## Data Retention

   - Oracle ORDERS table: keep online for 90 days post-cutover (legal hold)
   - Cosmos DB backups: 30 days (default Azure policy)
   - Archive old backups to Azure Archive tier after 30 days (cost optimization)
   ```

2. **Feature flag configuration** (`rollback-plan/feature-flags.yml`):
   ```yaml
   featureFlags:
     migration:
       orders:
         enabled: false
         description: "Route orders queries to Cosmos DB"
         rolloutPercentage: 0  # 0-100
         targetDate: "2026-04-22"
         owner: "DataEngineering"

     migration:
       order_items:
         enabled: false
         description: "Route order items queries to Cosmos DB"
         rolloutPercentage: 0
         targetDate: "2026-04-29"
         owner: "DataEngineering"
   ```

## Outputs

### Output 1: `migration/` Directory Structure

```
modernization-output/{slug}/migration/
├── infrastructure/
│   ├── terraform/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars.template
│   ├── bicep/
│   │   ├── main.bicep
│   │   └── parameters.json
│   └── azure-functions/
│       ├── host.json
│       ├── local.settings.json.template
│       ├── OrderAuditProcessor/
│       │   ├── function.json
│       │   └── __init__.py
│       └── ... (one per change feed processor)
├── transform-scripts/
│   ├── converters.py
│   ├── id_generator.py
│   ├── denormalization.py
│   ├── {table_name}.py (per-table transformer)
│   └── requirements.txt
├── export-queries/
│   ├── orders_export.sql
│   ├── order_items_export.sql
│   ├── customers_export.sql
│   └── ... (one per Oracle table)
├── adf-pipeline.json
├── validation/
│   ├── pre-migration.sql
│   ├── post-migration.sql
│   ├── reconciliation.py
│   └── sample-records.sql
├── seed-data/
│   ├── seed-orders.jsonl
│   ├── seed-customers.jsonl
│   └── README.md
├── rollback-plan.md
├── feature-flags.yml
├── migration-manifest.json
└── README.md
```

### Output 2: `migration-manifest.json` — Execution Checklist

```json
{
  "generated_at": "2026-03-22T15:00:00Z",
  "slug": "order-jsp",
  "target_system": "Azure Cosmos DB",
  "migration_phases": {
    "phase_1_infrastructure": {
      "status": "generated",
      "outputs": [
        "infrastructure/terraform/main.tf",
        "infrastructure/azure-functions/"
      ],
      "execution_notes": "Run: terraform plan && terraform apply"
    },
    "phase_2_data_transformation": {
      "status": "generated",
      "outputs": [
        "transform-scripts/orders.py",
        "transform-scripts/converters.py"
      ],
      "execution_notes": "Run: python transform-scripts/orders.py staging/orders.csv output/orders.jsonl"
    },
    "phase_3_adf_pipeline": {
      "status": "generated",
      "outputs": ["adf-pipeline.json"],
      "execution_notes": "Import into Azure Data Factory portal and trigger"
    },
    "phase_4_validation": {
      "status": "generated",
      "outputs": [
        "validation/pre-migration.sql",
        "validation/post-migration.sql"
      ],
      "execution_notes": "Run pre-migration before starting load, post-migration after completion"
    },
    "phase_5_rollback": {
      "status": "generated",
      "outputs": [
        "rollback-plan.md",
        "feature-flags.yml"
      ],
      "execution_notes": "Review rollback plan with team; keep feature flag OFF until validation complete"
    }
  },
  "tables_migrated": 5,
  "containers_created": 5,
  "total_transform_scripts": 5,
  "estimated_data_volume": "50 GB",
  "estimated_migration_time": "4 hours",
  "pre_migration_checklist": [
    "Oracle connectivity verified",
    "Cosmos DB provisioned (terraform apply successful)",
    "Azure Data Factory linked services configured",
    "Azure Blob Storage staging account created",
    "Feature flags deployed to application"
  ],
  "post_migration_checklist": [
    "Data reconciliation report shows 0 discrepancies",
    "Cosmos DB query performance within SLA",
    "Feature flag: migration.orders.enabled = false (rollout disabled)",
    "Monitoring and alerting configured",
    "Runbook reviewed with operations team"
  ]
}
```

### Output 3: `README.md` — Execution Guide

Comprehensive markdown guide explaining:
- Prerequisites (Azure subscriptions, credentials, network access)
- Step-by-step execution (terraform → ADF → validation → feature flag rollout)
- Troubleshooting common issues (throttling, type conversion errors, data mismatches)
- Monitoring dashboards and alert configuration
- References to rollback procedures

## Generation Principles

Encode these principles to ensure reliable migration:

1. **Preserve legacy IDs** — every document includes `legacyId` for audit trail and join back to Oracle during transition
2. **Immutable transformation** — transformation scripts are deterministic; re-running on same data produces same output
3. **Fail fast and log** — transformation scripts log every error with legacy ID and row number; failed rows written to dead-letter queue
4. **Validate early and often** — pre-migration queries run before ETL; reconciliation runs after to catch data loss/corruption
5. **Plan for rollback** — feature flags enable instant traffic switchback; blue-green setup allows testing in parallel with production
6. **Type safety** — all Oracle types explicitly converted; no implicit NULL handling or type mismatches
7. **Testable outputs** — seed-data/ contains sample records for unit testing transformation scripts before full migration

## Execution Checklist

Before marking migration generation complete, verify:

- [ ] All 5 phases executed
- [ ] Terraform or Bicep templates generate without errors
- [ ] Per-table transformation scripts created with type converters and denormalization logic
- [ ] Oracle export queries validated against schema-analysis.md
- [ ] Azure Data Factory pipeline JSON is valid and importable
- [ ] Pre-migration and post-migration validation queries provided
- [ ] Reconciliation script compares Oracle and Cosmos DB counts
- [ ] Rollback plan covers point-in-time restore, blue-green, and canary strategies
- [ ] Feature flag configuration provided for gradual rollout
- [ ] migration-manifest.json documents all generated artifacts
- [ ] README.md explains prerequisites, execution steps, and troubleshooting
- [ ] Seed data directory contains sample transformed records
- [ ] All scripts include headers with source table, target container, and timestamp
