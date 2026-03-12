# Skill: Detect Runtime Drift

## Description

Detect drift between the runtime versions in use in the repository and the approved runtime versions defined in the desired state specification.

## Inputs

| Input | Type | Required | Description |
|-------|------|----------|-------------|
| `build_files` | list[string] | Yes | Paths to build files (from `scan_repository`). |
| `dockerfiles` | list[string] | Yes | Paths to Dockerfiles (from `scan_repository`). |
| `ci_cd_files` | list[string] | Yes | Paths to CI/CD pipeline files (from `scan_repository`). |
| `runtime_configs` | list[string] | Yes | Paths to runtime configuration files (from `scan_repository`). |
| `desired_state` | object | Yes | Parsed content of `specs/desired_state.yml`. |

## Outputs

| Output | Type | Description |
|--------|------|-------------|
| `detected_runtimes` | list[DetectedRuntime] | All runtime versions detected across all scanned files. |
| `runtime_issues` | list[RuntimeIssue] | List of issues where detected runtime versions violate the desired state. |

### DetectedRuntime Object Schema

```json
{
  "runtime": "string",
  "version": "string",
  "source_file": "string",
  "detection_method": "string"
}
```

### RuntimeIssue Object Schema

```json
{
  "runtime": "string",
  "detected_version": "string",
  "minimum_allowed_version": "string",
  "source_file": "string",
  "severity": "string",
  "message": "string"
}
```

| Field | Description |
|-------|-------------|
| `severity` | One of: `critical`, `high`, `medium`, `low`, `info`. |
| `message` | Human-readable description of the drift. |

## Behavior

### Java Runtime Detection

1. Scan `pom.xml` for:
   - `<java.version>` property
   - `<maven.compiler.source>` / `<maven.compiler.target>`
   - `<maven.compiler.release>`
2. Scan `build.gradle` / `build.gradle.kts` for:
   - `sourceCompatibility`, `targetCompatibility`
   - `java { toolchain { languageVersion } }`
3. Scan Dockerfiles for `FROM eclipse-temurin:XX`, `FROM openjdk:XX`, `FROM amazoncorretto:XX` or similar base images.
4. Scan CI/CD files for `java-version` in setup-java actions or equivalent tool setup steps.
5. Scan `.java-version` or `.tool-versions` files.

### Node.js Runtime Detection

1. Scan `package.json` for the `engines.node` field.
2. Scan Dockerfiles for `FROM node:XX` base images.
3. Scan `.nvmrc` files.
4. Scan `.tool-versions` for `nodejs` entries.
5. Scan CI/CD files for `node-version` in setup-node actions.

### Python Runtime Detection

1. Scan `pyproject.toml` for `[tool.poetry] python` or `[project] requires-python`.
2. Scan `runtime.txt` for Heroku-style runtime pins.
3. Scan Dockerfiles for `FROM python:XX` base images.
4. Scan `.tool-versions` for `python` entries.
5. Scan CI/CD files for `python-version` in setup-python actions.

### Drift Evaluation

For each detected runtime and version:

1. Look up the runtime in `desired_state.runtime`.
2. Compare the detected version against `minimum_version` using semantic versioning.
3. Check whether the version appears in `deprecated_versions`.
4. Assign severity:
   - `critical` — version listed in `deprecated_versions` with known CVEs.
   - `high` — version below `minimum_version`.
   - `medium` — version is not in `allowed_versions` but above minimum.
   - `info` — version is compliant.
5. Emit a `RuntimeIssue` for every non-compliant finding.

## Constraints

- Parse version strings leniently; handle formats like `11`, `11.0`, `11.0.2`, `java11`, `temurin-17`.
- If version cannot be determined from a source, emit an `info`-severity issue noting that the version is unpinned.
- Do not make network calls.

## Example Output

```json
{
  "detected_runtimes": [
    {
      "runtime": "java",
      "version": "11",
      "source_file": "pom.xml",
      "detection_method": "maven.compiler.source property"
    },
    {
      "runtime": "java",
      "version": "11",
      "source_file": "Dockerfile",
      "detection_method": "FROM eclipse-temurin:11 base image"
    }
  ],
  "runtime_issues": [
    {
      "runtime": "java",
      "detected_version": "11",
      "minimum_allowed_version": "17",
      "source_file": "pom.xml",
      "severity": "high",
      "message": "Java 11 is below the minimum allowed version of 17. Upgrade to Java 17 or 21."
    },
    {
      "runtime": "java",
      "detected_version": "11",
      "minimum_allowed_version": "17",
      "source_file": "Dockerfile",
      "severity": "high",
      "message": "Dockerfile uses eclipse-temurin:11 which is below the minimum allowed Java version of 17."
    }
  ]
}
```
