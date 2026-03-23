# Scan Orphans — Codebase-Wide Unclaimed Code Detection

Scan the entire codebase for files and components NOT discovered by any entry-point analysis. This is the safety net that catches everything the entry-point-driven approach inherently cannot reach.

## When to Run
Run this AFTER you've completed `/modernize` (or `/analyze-entry`) for ALL known entry points. This is a codebase-wide sweep — it compares the entire codebase against the union of all analyzed files.

## Arguments
- `$ARGUMENTS` — optional: path to the codebase root (defaults to current working directory)

## Execution Flow

### Step 1: Gather All Fingerprints
1. Find all `fingerprint.json` files under `modernization-output/*/`
2. Build the "claimed files" set — union of all file paths across all fingerprints
3. Report: "{N} entry points analyzed, {M} total files claimed"
4. If no fingerprint files exist, STOP — run `/modernize` or `/analyze-entry` first

### Step 2: Run Orphan Scanner
Run the **scan-orphans** skill (orphan-scanner):
- Input: All fingerprint.json files + the complete codebase
- Output: `modernization-output/orphan-report.md`

### Step 3: Report Results

```
## Orphan Scan Complete

### Summary
- Entry points analyzed: {count of slug directories}
- Total codebase files: {count}
- Files claimed by analyses: {count}
- Unclaimed files: {count}
  - 🔴 Active orphans: {count}
  - 🟡 Uncertain: {count}
  - 🟢 Dead code: {count}

### Report
See: modernization-output/orphan-report.md

### Recommended Actions
{If active orphans found}:
  - Run `/modernize <path> <service-name>` for each active orphan to bring it into the pipeline
  - After addressing active orphans, re-run `/scan-orphans` to confirm full coverage
{If no active orphans}:
  - ✅ Full codebase coverage achieved — all active code has been analyzed
```

## Error Handling
- If no fingerprint.json files exist → stop with guidance to run /modernize first
- If codebase path doesn't exist → report error
- If scan finds hundreds of orphans → likely the codebase root is wrong or not enough entry points have been analyzed; warn the user
