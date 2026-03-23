# Modernize Entry Point — End-to-End Pipeline

Run the full modernization pipeline for a single legacy entry point: analyze → funcspec → techspec → JIRA stories → code generation → completeness verification.

## Arguments
- `$ARGUMENTS` — expects: `<entry-point-path> <service-name>`
  - **entry-point-path**: Path to the legacy file to start analysis from (e.g., `src/main/webapp/WEB-INF/views/order.jsp`)
  - **service-name**: Name of the microservice to generate (e.g., `order-service`). Becomes the Maven artifact ID, base package suffix, Docker image name.

## Execution Flow

Parse the arguments:
```
entry_point = first argument (file path)
service_name = second argument (microservice name)
slug = derive from entry-point name (lowercase, hyphens, e.g., "order-jsp", "payment-service")
output_dir = modernization-output/{slug}/
```

### Step 1: Setup
1. Create the output directory: `modernization-output/{slug}/`
2. Confirm the entry-point file exists on disk
3. If the file doesn't exist, STOP and report the error — do not proceed

### Step 2: Analyze Entry Point
Run the **analyze-entry** skill (code-analysis):
- Input: the entry-point file path
- Outputs saved to `{output_dir}/`:
  - `analysis.md` — component inventory, behavioral inventory, data flow
  - `fingerprint.json` — source file hashes and signatures
  - `source-extracts.md` — exhaustive raw technical extraction

**Validation gate**: Confirm all 3 files were created and are non-empty. If analysis found zero connected components, warn but continue.

### Step 3: Update Contract Registry
After analysis completes:
- Read `{output_dir}/analysis.md` for shared resources (Kafka topics, DB tables, API endpoints)
- Append to `modernization-output/contracts.json`
- If `contracts.json` doesn't exist yet, create it with the initial structure

### Step 4: Generate Functional Specification
Run the **gen-funcspec** skill (funcspec-writer):
- Input: `{output_dir}/analysis.md` + `{output_dir}/source-extracts.md`
- Output: `{output_dir}/funcspec.md`

**Validation gate**: Confirm funcspec.md was created and contains at least one FR-xxx requirement. If it would be empty, STOP and ask for clarification.

### Step 5: Generate Technical Specification
Run the **gen-techspec** skill (techspec-writer):
- Input: `{output_dir}/analysis.md` + `{output_dir}/source-extracts.md` + `{output_dir}/funcspec.md`
- Output: `{output_dir}/techspec.md`
- **IMPORTANT**: Do NOT re-read legacy source code. source-extracts.md already captured everything.

**Validation gate**: Confirm techspec.md was created and contains the 13 required sections.

### Step 6: Generate JIRA Stories
Run the **gen-jira** skill (jira-modernize):
- Input: `{output_dir}/analysis.md` + `{output_dir}/funcspec.md` + `{output_dir}/techspec.md` + `{output_dir}/fingerprint.json`
- Output: `{output_dir}/jira-stories.md`

**Validation gate**: Confirm jira-stories.md was created and contains at least one story with traceability header.

### Step 7: Generate Modernized Code
Run the **gen-code** skill (code-generator):
- Input: All upstream artifacts + `modernization-output/contracts.json`
- Arguments: `{slug} {service_name}`
- Outputs:
  - `{output_dir}/{service_name}/` — complete Spring Boot 4.x project
  - `{output_dir}/code-manifest.json` — legacy-to-modern file mapping

**Validation gate**: Confirm the project directory was created with at least a pom.xml/build.gradle and src/ structure. Confirm code-manifest.json exists.

### Step 8: Verify Completeness
Run the **verify-completeness** skill (completeness-checker):
- Input: All upstream artifacts + the generated code at `{output_dir}/{service_name}/`
- Output: `{output_dir}/completeness-report.md`

### Step 9: Report Results
After all steps complete, output a summary:

```
## Modernization Complete: {slug}

### Artifacts Generated
| Step | Artifact | Path | Status |
|------|----------|------|--------|
| 1 | Analysis | {output_dir}/analysis.md | ✅ |
| 1 | Fingerprint | {output_dir}/fingerprint.json | ✅ |
| 1 | Source Extracts | {output_dir}/source-extracts.md | ✅ |
| 2 | Functional Spec | {output_dir}/funcspec.md | ✅ |
| 3 | Technical Spec | {output_dir}/techspec.md | ✅ |
| 4 | JIRA Stories | {output_dir}/jira-stories.md | ✅ |
| 5 | Generated Code | {output_dir}/{service_name}/ | ✅ |
| 5 | Code Manifest | {output_dir}/code-manifest.json | ✅ |
| 6 | Completeness Report | {output_dir}/completeness-report.md | ✅ |

### Completeness Summary
{Extract the overall completeness percentage and status from completeness-report.md}

### Next Steps
1. Review the completeness report for any gaps
2. Review JIRA stories and adjust sizing/priority
3. Run `/check-drift {slug}` periodically to detect legacy changes
4. After all entry points are analyzed, run `/scan-orphans` for full coverage
```

## Error Handling
- If any step fails, report which step failed and what went wrong
- Do NOT skip steps — the pipeline is sequential and each step depends on prior outputs
- If a validation gate fails, STOP the pipeline and report the issue
- If contracts.json shows a shared resource conflict, warn and ask for confirmation before proceeding to code generation
