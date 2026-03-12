# Workflow: Analyze Repository

## Description

Orchestrate the full technical debt analysis of a repository by executing all analysis skills in sequence. This workflow uses the desired state specification as the source of truth and produces a comprehensive technical debt report.

## Trigger

This workflow can be triggered:
- Manually via GitHub Copilot agent invocation.
- On a scheduled basis (e.g., weekly).
- On push or pull request events targeting the default branch.

## Inputs

| Input | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `repository_path` | string | No | `.` (current repository root) | Path to the repository to analyze. |
| `desired_state_path` | string | No | `specs/desired_state.yml` | Path to the desired state specification file. |
| `output_path` | string | No | `outputs/tech_debt_report.md` | Path where the generated report should be written. |
| `fail_on_critical` | boolean | No | `true` | If `true`, the workflow exits with a non-zero status when critical issues are found. |
| `fail_on_high` | boolean | No | `false` | If `true`, the workflow exits with a non-zero status when high-severity issues are found. |

## Steps

---

### Step 1: Load Desired State

**Action:** Read and parse the desired state specification file at `desired_state_path`.

**Input:** `desired_state_path`

**Output:** `desired_state` (parsed object)

**Error handling:**
- If the file does not exist, abort the workflow and emit error: `"Desired state specification not found at {desired_state_path}. Cannot proceed with analysis."`
- If the file is malformed YAML, abort and emit error: `"Failed to parse desired state specification: {parse_error}."`

---

### Step 2: Scan Repository

**Skill:** [`scan_repository`](../skills/scan_repository.md)

**Input:**
- `repository_path`

**Output:**
- `build_files`
- `dockerfiles`
- `ci_cd_files`
- `runtime_configs`
- `source_structure`

**Error handling:**
- If `repository_path` does not exist or is not accessible, abort and emit error.
- Emit a warning if no build files are found, but continue.

---

### Step 3: Extract Dependencies

**Skill:** [`extract_dependencies`](../skills/extract_dependencies.md)

**Input:**
- `build_files` (from Step 2)
- `repository_path`

**Output:**
- `dependencies`

**Error handling:**
- If a build file cannot be parsed, log a warning for that file and continue with remaining build files.
- Emit warning if `dependencies` list is empty.

---

### Step 4: Detect Runtime Drift

**Skill:** [`detect_runtime_drift`](../skills/detect_runtime_drift.md)

**Input:**
- `build_files` (from Step 2)
- `dockerfiles` (from Step 2)
- `ci_cd_files` (from Step 2)
- `runtime_configs` (from Step 2)
- `desired_state` (from Step 1)

**Output:**
- `detected_runtimes`
- `runtime_issues`

**Error handling:**
- Non-fatal: log warnings for files that cannot be read. Continue with remaining files.

---

### Step 5: Detect Vulnerabilities

**Skill:** [`detect_vulnerabilities`](../skills/detect_vulnerabilities.md)

**Input:**
- `dependencies` (from Step 3)
- `desired_state` (from Step 1)

**Output:**
- `vulnerabilities`
- `security_issues`
- `banned_library_issues`

**Error handling:**
- If an advisory source API is unavailable, log a warning and continue with remaining sources.
- Do not abort if all advisory sources are unavailable; emit a prominent warning in the report noting incomplete security data.

---

### Step 6: Detect Framework and Platform Drift

**Skill:** [`detect_framework_drift`](../skills/detect_framework_drift.md)

**Input:**
- `dependencies` (from Step 3)
- `build_files` (from Step 2)
- `ci_cd_files` (from Step 2)
- `desired_state` (from Step 1)
- `repository_path`

**Output:**
- `framework_issues`
- `build_tool_issues`
- `platform_drift_issues`

**Error handling:**
- Non-fatal: log warnings for unreadable files. Continue with remaining files.

---

### Step 7: Generate Report

**Skill:** [`generate_report`](../skills/generate_report.md)

**Input:**
- `repository_path`
- `desired_state`
- `runtime_issues` (from Step 4)
- `vulnerabilities` (from Step 5)
- `security_issues` (from Step 5)
- `banned_library_issues` (from Step 5)
- `framework_issues` (from Step 6)
- `build_tool_issues` (from Step 6)
- `platform_drift_issues` (from Step 6)
- `dependencies` (from Step 3)
- `report_date` (current UTC date in ISO 8601 format)

**Output:**
- `report_path`
- `issue_summary`

**Error handling:**
- If the output directory does not exist, create it before writing.
- If writing fails, abort with an error.

---

### Step 8: AI Summary (Optional)

**Skill:** [`summarize_with_claude`](../skills/summarize_with_claude.md)

**Condition:** Only executed when the `ANTHROPIC_API_KEY` environment variable is set.

**Input:**
- `issue_summary` (from Step 7)
- `runtime_issues` (from Step 4)
- `vulnerabilities` (from Step 5)
- `security_issues` (from Step 5)
- `banned_library_issues` (from Step 5)
- `framework_issues` (from Step 6)
- `build_tool_issues` (from Step 6)
- `platform_drift_issues` (from Step 6)
- `repository_name` (derived from `repository_path`)
- `report_date` (current UTC date in ISO 8601 format)

**Output:**
- `ai_summary`
- `risk_rating`
- `top_priorities`
- `model_used`

**Error handling:**
- If `ANTHROPIC_API_KEY` is not set, skip this step silently and continue to Step 9.
- If the Claude API call fails, log a warning and continue to Step 9 without an AI summary.

---

### Step 9: Evaluate Exit Condition

After generating the report, evaluate the following conditions using `issue_summary`:

1. If `fail_on_critical` is `true` and `issue_summary.by_severity.critical > 0`:
   - Emit error: `"Analysis failed: {N} critical issue(s) found. See report at {output_path}."`
   - Exit with status code `2`.

2. If `fail_on_high` is `true` and `issue_summary.by_severity.high > 0`:
   - Emit error: `"Analysis failed: {N} high-severity issue(s) found. See report at {output_path}."`
   - Exit with status code `1`.

3. If no failure conditions are triggered:
   - Emit success: `"Analysis complete. {total} issue(s) found. Report written to {output_path}."`
   - Exit with status code `0`.

---

## Execution Diagram

```
┌─────────────────────────────────┐
│    Step 1: Load Desired State   │
└────────────────┬────────────────┘
                 │
┌────────────────▼────────────────┐
│    Step 2: Scan Repository      │
└────────────────┬────────────────┘
                 │
┌────────────────▼────────────────┐
│  Step 3: Extract Dependencies   │
└────┬───────────┴────────────────┘
     │           │
     │    ┌──────▼──────────────────────┐
     │    │ Step 4: Detect Runtime Drift│
     │    └──────┬──────────────────────┘
     │           │
     │    ┌──────▼──────────────────────────┐
     │    │ Step 5: Detect Vulnerabilities  │
     │    └──────┬──────────────────────────┘
     │           │
     │    ┌──────▼──────────────────────────────────┐
     │    │ Step 6: Detect Framework & Platform Drift│
     │    └──────┬──────────────────────────────────┘
     │           │
┌────▼───────────▼────────────────┐
│    Step 7: Generate Report      │
└────────────────┬────────────────┘
                 │
┌────────────────▼────────────────────┐
│ Step 8: AI Summary (Claude)         │
│         [optional — skipped if      │
│          ANTHROPIC_API_KEY not set] │
└────────────────┬────────────────────┘
                 │
┌────────────────▼────────────────┐
│  Step 9: Evaluate Exit Condition│
└─────────────────────────────────┘
```

## Outputs

| Output | Description |
|--------|-------------|
| `outputs/tech_debt_report.md` | Full technical debt report in Markdown format, including an AI summary section when Claude is available. |
| Exit code `0` | Analysis succeeded; no threshold-breaching issues found. |
| Exit code `1` | High-severity issues found (when `fail_on_high` is `true`). |
| Exit code `2` | Critical issues found (when `fail_on_critical` is `true`). |

## Notes

- All steps must complete before the report is generated.
- Steps 4, 5, and 6 are independent of each other once Step 3 completes, and may be executed in parallel if the agent supports concurrent skill execution.
- This workflow does not modify any files in the analyzed repository.
