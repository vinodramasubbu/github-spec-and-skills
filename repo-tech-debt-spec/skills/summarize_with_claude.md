# Skill: Summarize with Claude

## Description

Use the Claude API (Anthropic) to generate an intelligent, narrative AI summary of the technical debt analysis findings. This skill produces a human-readable executive briefing, a risk assessment, and prioritized remediation guidance that goes beyond the structured tables in the report — synthesizing patterns, highlighting root causes, and explaining trade-offs in plain language.

## Inputs

| Input | Type | Required | Description |
|-------|------|----------|-------------|
| `issue_summary` | object | Yes | Machine-readable issue count summary from `generate_report` (`by_severity`, `by_category`, `total`). |
| `runtime_issues` | list[RuntimeIssue] | Yes | Output from `detect_runtime_drift`. |
| `vulnerabilities` | list[Vulnerability] | Yes | Output from `detect_vulnerabilities`. |
| `security_issues` | list[SecurityIssue] | Yes | Output from `detect_vulnerabilities`. |
| `banned_library_issues` | list[BannedLibraryIssue] | Yes | Output from `detect_vulnerabilities`. |
| `framework_issues` | list[FrameworkIssue] | Yes | Output from `detect_framework_drift`. |
| `build_tool_issues` | list[BuildToolIssue] | Yes | Output from `detect_framework_drift`. |
| `platform_drift_issues` | list[PlatformDriftIssue] | Yes | Output from `detect_framework_drift`. |
| `repository_name` | string | Yes | Name of the repository being analyzed. |
| `report_date` | string | Yes | ISO 8601 date string of the analysis run. |
| `claude_model` | string | No | Claude model to use. Defaults to `claude-3-5-sonnet-20241022`. |
| `max_tokens` | number | No | Maximum tokens for the Claude response. Defaults to `2048`. |

## Outputs

| Output | Type | Description |
|--------|------|-------------|
| `ai_summary` | string | Full Claude-generated narrative in Markdown format. |
| `risk_rating` | string | Overall risk rating derived by Claude: `critical`, `high`, `medium`, `low`, or `none`. |
| `top_priorities` | list[string] | Ordered list of the top 5 remediation actions Claude recommends (plain text, one action per item). |
| `model_used` | string | The Claude model identifier that produced the response. |

## Behavior

### 1. Build the Prompt

Construct a structured system prompt and user message to send to Claude.

**System prompt:**

```
You are a senior software engineer and security expert reviewing a technical debt analysis report.
Your job is to synthesize the findings into a clear, actionable executive briefing for an engineering team.
Be concise, specific, and practical. Use Markdown formatting in your response.
```

**User message template:**

```
Repository: {repository_name}
Analysis Date: {report_date}

## Issue Summary
- Total Issues: {issue_summary.total}
- Critical: {issue_summary.by_severity.critical}
- High: {issue_summary.by_severity.high}
- Medium: {issue_summary.by_severity.medium}
- Low: {issue_summary.by_severity.low}

## Runtime Issues ({count})
{runtime_issues serialized as a compact JSON array, or "None" if empty}

## Security Vulnerabilities ({count})
{vulnerabilities serialized as a compact JSON array, or "None" if empty}

## Banned Libraries ({count})
{banned_library_issues serialized as a compact JSON array, or "None" if empty}

## Framework Issues ({count})
{framework_issues serialized as a compact JSON array, or "None" if empty}

## Build Tool Issues ({count})
{build_tool_issues serialized as a compact JSON array, or "None" if empty}

## Platform Drift Issues ({count})
{platform_drift_issues serialized as a compact JSON array, or "None" if empty}

Please provide:
1. **Executive Briefing** (2–3 paragraphs): Summarize the overall health of this repository, highlighting the most significant risks and any patterns you observe across the findings.
2. **Risk Rating**: Rate the overall repository risk as one of: `critical`, `high`, `medium`, `low`, or `none`. State the rating clearly on its own line as: `Risk Rating: <rating>`.
3. **Top 5 Remediation Priorities**: A numbered list of the five most important actions the team should take, ordered by urgency. Each item should name the specific issue and the concrete action required.
```

### 2. Call the Claude API

Send the prompt to the Claude Messages API:

- **Endpoint:** `POST https://api.anthropic.com/v1/messages`
- **Headers:**
  - `x-api-key`: value from the `ANTHROPIC_API_KEY` environment variable
  - `anthropic-version`: `2023-06-01`
  - `content-type`: `application/json`
- **Request body:**
  ```json
  {
    "model": "{claude_model}",
    "max_tokens": {max_tokens},
    "system": "{system_prompt}",
    "messages": [
      { "role": "user", "content": "{user_message}" }
    ]
  }
  ```

### 3. Parse the Response

1. Extract the text content from `response.content[0].text`.
2. Parse the `Risk Rating: <rating>` line from the response to populate `risk_rating`. Use a case-insensitive regex match: `Risk Rating:\s*(critical|high|medium|low|none)`.
3. Parse the numbered list under "Top 5 Remediation Priorities" to populate `top_priorities` (strip numbering, trim whitespace).
4. Store the full response text as `ai_summary`.
5. Record the model identifier from `response.model` as `model_used`.

### 4. Append to Report

Append the `ai_summary` to `outputs/tech_debt_report.md` under a new section:

```markdown
---

## AI Analysis (Claude)

> *Generated by {model_used} on {report_date}. AI summaries are advisory and should be reviewed by a qualified engineer.*

{ai_summary}
```

## Constraints

- The `ANTHROPIC_API_KEY` environment variable must be set. If it is missing or empty, skip this skill gracefully and log a warning: `"ANTHROPIC_API_KEY not set; skipping Claude AI summary."` Do not abort the workflow.
- Truncate individual issue lists to a maximum of 20 items before inserting into the prompt to stay within token limits. If truncated, append a note: `"... and {N} more"`.
- Do not include raw file contents or source code in the prompt.
- If the Claude API returns a non-2xx response or times out after 30 seconds, log a warning and set `ai_summary` to `null`. Do not retry more than once.
- The `risk_rating` field must be one of the five allowed values. If Claude's response does not contain a parseable rating, default to `"unknown"`.
- This skill must not modify any source files in the analyzed repository.

## Example Output

```json
{
  "ai_summary": "## Executive Briefing\n\nThis repository presents a **high overall risk** ...",
  "risk_rating": "high",
  "top_priorities": [
    "Upgrade jackson-databind from 2.13.4 to 2.14.0 or later to resolve multiple critical CVEs.",
    "Migrate from log4j to SLF4J + Logback immediately to eliminate the Log4Shell attack surface.",
    "Upgrade Spring Boot from 2.7.18 to 3.2.x to address the high-severity DoS vulnerability and reach a supported version.",
    "Pin Node.js runtime to version 20 LTS in .nvmrc and Dockerfile; version 16 is end-of-life.",
    "Add a security-scan step to the GitHub Actions workflow to enforce vulnerability checks on every pull request."
  ],
  "model_used": "claude-3-5-sonnet-20241022"
}
```
