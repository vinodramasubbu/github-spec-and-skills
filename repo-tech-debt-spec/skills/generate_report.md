# Skill: Generate Report

## Description

Aggregate all findings from the analysis skills and produce a structured, human-readable technical debt report in Markdown format.

## Inputs

| Input | Type | Required | Description |
|-------|------|----------|-------------|
| `repository_path` | string | Yes | Path to the repository root. |
| `desired_state` | object | Yes | Parsed desired state specification. |
| `runtime_issues` | list[RuntimeIssue] | Yes | Output from `detect_runtime_drift`. |
| `vulnerabilities` | list[Vulnerability] | Yes | Output from `detect_vulnerabilities`. |
| `security_issues` | list[SecurityIssue] | Yes | Output from `detect_vulnerabilities`. |
| `banned_library_issues` | list[BannedLibraryIssue] | Yes | Output from `detect_vulnerabilities`. |
| `framework_issues` | list[FrameworkIssue] | Yes | Output from `detect_framework_drift`. |
| `build_tool_issues` | list[BuildToolIssue] | Yes | Output from `detect_framework_drift`. |
| `platform_drift_issues` | list[PlatformDriftIssue] | Yes | Output from `detect_framework_drift`. |
| `dependencies` | list[Dependency] | Yes | Full dependency list from `extract_dependencies`. |
| `report_date` | string | Yes | ISO 8601 date string for the report generation timestamp. |

## Outputs

| Output | Type | Description |
|--------|------|-------------|
| `report_path` | string | Path where the generated report is written: `outputs/tech_debt_report.md`. |
| `issue_summary` | object | Machine-readable summary of issue counts by type and severity. |

### IssueSummary Object Schema

```json
{
  "total": "number",
  "by_severity": {
    "critical": "number",
    "high": "number",
    "medium": "number",
    "low": "number",
    "info": "number"
  },
  "by_category": {
    "runtime": "number",
    "dependency": "number",
    "security": "number",
    "framework": "number",
    "build_tool": "number",
    "platform": "number"
  }
}
```

## Behavior

### Report Structure

Generate a Markdown document written to `outputs/tech_debt_report.md` with the following sections in order:

---

#### 1. Header

Include:
- Report title: `# Technical Debt Report`
- Repository name (derived from the repository path).
- Report generation date.
- Desired state specification file reference.

#### 2. Executive Summary

Include:
- Total number of issues found.
- Breakdown by severity: critical, high, medium, low, info.
- Breakdown by category: Runtime, Dependency, Security, Framework, Build Tool, Platform Drift.
- A one-paragraph overall assessment.

Use a Markdown table for the severity breakdown and a second table for category breakdown.

#### 3. Runtime Issues

For each `RuntimeIssue`:
- Detected runtime and version.
- Minimum allowed version per desired state.
- Source file where the runtime version was detected.
- Severity badge.
- Recommended action.

Render as a Markdown table. If no runtime issues are found, render a ✅ green checkmark with the message "All runtimes are compliant with the desired state."

#### 4. Dependency Issues

For each dependency flagged as outdated (version below allowed or age exceeding maximum):
- Dependency name, group, and detected version.
- Reason flagged (outdated, banned, exceeded age limit).
- Source file.
- Severity.
- Recommended action.

Render as a Markdown table. If no dependency issues are found, render a ✅ message.

#### 5. Security Vulnerabilities

For each `Vulnerability`:
- CVE ID / GHSA ID / OSV ID.
- Affected dependency and version.
- Severity and CVSS score (if available).
- Summary of the vulnerability.
- Fixed-in version (if known).
- Advisory source.

Render as a Markdown table sorted by CVSS score descending. Include `BannedLibraryIssues` at the top of this section.

If no vulnerabilities are found, render a ✅ message.

#### 6. Framework Issues

For each `FrameworkIssue`:
- Framework name and detected version.
- Minimum allowed version.
- Source file.
- Severity.
- Recommended action (upgrade path).

Render as a Markdown table. If no framework issues, render ✅.

#### 7. Build Tool Issues

For each `BuildToolIssue`:
- Tool name and detected version.
- Minimum allowed version.
- Source file.
- Severity.
- Recommended action.

Render as a Markdown table. If no build tool issues, render ✅.

#### 8. Platform Drift

For each `PlatformDriftIssue`:
- Check name.
- Expected value.
- Actual value found.
- Severity.
- Recommended action.

Render as a Markdown table. If no platform drift issues, render ✅.

#### 9. Recommended Remediation

Generate a prioritized, numbered list of remediation actions ordered by severity (critical first):

1. Group related issues into a single action item where possible.
2. Each action item must include:
   - **Issue**: Brief description.
   - **Action**: Specific upgrade or configuration change required.
   - **Priority**: Derived from severity.
   - **Effort**: Estimated effort level: `Low`, `Medium`, `High`.

#### 10. Appendix: Full Dependency List

Render the complete dependency list as a collapsible `<details>` Markdown block containing a table with: name, group, version, scope, source file, ecosystem.

---

### Severity Badges

Use inline Markdown emoji for severity:
- `critical` → 🔴
- `high` → 🟠
- `medium` → 🟡
- `low` → 🔵
- `info` → ⚪

### Issue Summary Computation

After generating the report:
1. Count all issues across all categories.
2. Aggregate counts by severity and category.
3. Return the `issue_summary` object.

## Constraints

- The report must be deterministic: given the same inputs, it must always produce the same output.
- Do not include placeholder sections for categories with no issues (except for "Recommended Remediation" and "Executive Summary").
- Dates must be formatted as `YYYY-MM-DD`.
- All Markdown tables must have a header row and use standard pipe syntax.
- Write the report file atomically (write to a temp path, then rename).

## Example Output (Excerpt)

```markdown
# Technical Debt Report

**Repository:** my-service  
**Generated:** 2024-11-15  
**Desired State:** specs/desired_state.yml  

---

## Executive Summary

| Severity | Count |
|----------|-------|
| 🔴 Critical | 2 |
| 🟠 High | 3 |
| 🟡 Medium | 1 |
| 🔵 Low | 0 |
| ⚪ Info | 0 |

**Total Issues: 6**

...

## Security Vulnerabilities

| CVE / Advisory | Dependency | Version | Severity | CVSS | Summary | Fixed In |
|---------------|------------|---------|----------|------|---------|---------|
| CVE-2023-20883 | spring-boot-starter-web | 2.7.18 | 🟠 High | 7.5 | Spring Boot DoS vulnerability | 2.7.12 |
```
