# Skill: Detect Vulnerabilities

## Description

Detect known security vulnerabilities in the repository's dependencies by cross-referencing them against public vulnerability databases defined in the desired state specification.

## Inputs

| Input | Type | Required | Description |
|-------|------|----------|-------------|
| `dependencies` | list[Dependency] | Yes | Normalized dependency list from the `extract_dependencies` skill. |
| `desired_state` | object | Yes | Parsed content of `specs/desired_state.yml`, specifically the `dependencies.security` section. |

## Outputs

| Output | Type | Description |
|--------|------|-------------|
| `vulnerabilities` | list[Vulnerability] | All CVEs or advisories found for the repository's dependencies. |
| `security_issues` | list[SecurityIssue] | Issues where vulnerability counts exceed the thresholds defined in the desired state. |
| `banned_library_issues` | list[BannedLibraryIssue] | Dependencies that match entries in `desired_state.dependencies.banned_libraries`. |

### Vulnerability Object Schema

```json
{
  "dependency_name": "string",
  "dependency_version": "string",
  "ecosystem": "string",
  "cve_id": "string | null",
  "ghsa_id": "string | null",
  "osv_id": "string | null",
  "severity": "string",
  "cvss_score": "number | null",
  "summary": "string",
  "fixed_in_version": "string | null",
  "advisory_source": "string"
}
```

### SecurityIssue Object Schema

```json
{
  "severity": "string",
  "count": "number",
  "threshold": "number",
  "message": "string"
}
```

### BannedLibraryIssue Object Schema

```json
{
  "dependency_name": "string",
  "detected_version": "string",
  "source_file": "string",
  "reason": "string",
  "severity": "string"
}
```

## Behavior

### Banned Library Check

1. For each dependency in the input list, check whether `name` matches any entry in `desired_state.dependencies.banned_libraries`.
2. If a `versions_below` constraint exists on the banned entry, only flag the dependency if its version is below that threshold.
3. Emit a `BannedLibraryIssue` with severity `critical` for any match.

### Vulnerability Advisory Lookup

Use the advisory sources listed in `desired_state.dependencies.security.advisory_sources`. Apply the following strategy for each source:

#### GitHub Advisory Database (GHSA)

- Query the GitHub Advisory Database REST API: `GET /advisories?ecosystem={ecosystem}&package={name}&version={version}`
- Map each advisory to a `Vulnerability` object using its `ghsa_id`, associated `cve_id`, and `severity`.

#### OSV (Open Source Vulnerabilities)

- Query the OSV REST API: `POST https://api.osv.dev/v1/query` with payload:
  ```json
  { "version": "{version}", "package": { "name": "{name}", "ecosystem": "{ecosystem}" } }
  ```
- Map each OSV record to a `Vulnerability` object.

#### NVD (National Vulnerability Database)

- Query the NVD CVE API: `GET https://services.nvd.nist.gov/rest/json/cves/2.0?cpeName=...`
- Use CPE matching for Maven (`cpe:2.3:a:{group}:{name}:{version}`) and npm packages.

### Threshold Evaluation

After collecting all vulnerabilities:

1. Count findings by severity: `critical`, `high`, `medium`, `low`.
2. Compare each count to the corresponding threshold in `desired_state.dependencies.security.vulnerability_threshold`.
3. Emit a `SecurityIssue` for any severity category that exceeds its threshold.

### Dependency Age Check

1. For each dependency with a resolved version, query the package registry to determine the release date of the declared version.
   - Maven: query Maven Central REST API.
   - npm: query `https://registry.npmjs.org/{name}`.
   - PyPI: query `https://pypi.org/pypi/{name}/json`.
2. Calculate the age in days from release date to the current date.
3. If age exceeds `desired_state.dependencies.maximum_dependency_age_days`, emit a `Vulnerability` entry with severity `medium` and advisory source `age-check`.

## Constraints

- Cache advisory API responses to avoid duplicate queries for the same package/version.
- If an advisory source is unavailable (network error or rate limit), log a warning and continue with remaining sources.
- Do not fail the entire skill if one advisory source is unreachable; mark affected results as `advisory_source: "unavailable"`.
- Only scan `is_direct: true` dependencies for age checks; transitive dependency age is informational only.

## Example Output

```json
{
  "vulnerabilities": [
    {
      "dependency_name": "spring-boot-starter-web",
      "dependency_version": "2.7.18",
      "ecosystem": "maven",
      "cve_id": "CVE-2023-20883",
      "ghsa_id": "GHSA-xjhv-p3fq-x59p",
      "osv_id": "GHSA-xjhv-p3fq-x59p",
      "severity": "high",
      "cvss_score": 7.5,
      "summary": "Spring Boot denial of service via specially crafted Spring MVC or Spring WebFlux application.",
      "fixed_in_version": "2.7.12",
      "advisory_source": "GitHub Advisory Database"
    }
  ],
  "security_issues": [
    {
      "severity": "high",
      "count": 1,
      "threshold": 0,
      "message": "1 high-severity vulnerability detected. Threshold is 0. Immediate remediation required."
    }
  ],
  "banned_library_issues": [
    {
      "dependency_name": "jackson-databind",
      "detected_version": "2.13.4",
      "source_file": "pom.xml",
      "reason": "Multiple CVEs in older versions. Upgrade to 2.14.0 or later.",
      "severity": "critical"
    }
  ]
}
```
