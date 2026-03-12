# Skill: Detect Framework Drift

## Description

Detect drift between the frameworks and build tools in use in the repository and the approved versions defined in the desired state specification. Also detect platform-level configuration violations.

## Inputs

| Input | Type | Required | Description |
|-------|------|----------|-------------|
| `dependencies` | list[Dependency] | Yes | Normalized dependency list from the `extract_dependencies` skill. |
| `build_files` | list[string] | Yes | Paths to build files identified by `scan_repository`. |
| `ci_cd_files` | list[string] | Yes | Paths to CI/CD pipeline files identified by `scan_repository`. |
| `desired_state` | object | Yes | Parsed content of `specs/desired_state.yml`. |
| `repository_path` | string | Yes | Root path of the repository. |

## Outputs

| Output | Type | Description |
|--------|------|-------------|
| `framework_issues` | list[FrameworkIssue] | Framework version violations relative to the desired state. |
| `build_tool_issues` | list[BuildToolIssue] | Build tool version violations relative to the desired state. |
| `platform_drift_issues` | list[PlatformDriftIssue] | Platform configuration violations relative to the desired state. |

### FrameworkIssue Object Schema

```json
{
  "framework": "string",
  "detected_version": "string",
  "minimum_allowed_version": "string",
  "source_file": "string",
  "severity": "string",
  "message": "string"
}
```

### BuildToolIssue Object Schema

```json
{
  "tool": "string",
  "detected_version": "string | null",
  "minimum_allowed_version": "string",
  "source_file": "string",
  "severity": "string",
  "message": "string"
}
```

### PlatformDriftIssue Object Schema

```json
{
  "check": "string",
  "expected": "string",
  "actual": "string | null",
  "severity": "string",
  "message": "string"
}
```

## Behavior

### Framework Detection

#### Spring Boot

1. Search `dependencies` list for artifacts with `group` = `org.springframework.boot` and `name` = `spring-boot-starter` (or `spring-boot-starter-*`).
2. Use the declared version (or BOM-managed version from `spring-boot-dependencies` BOM entry).
3. Compare against `desired_state.frameworks.spring_boot.minimum_version`.
4. Flag versions matching any pattern in `desired_state.frameworks.spring_boot.deprecated_versions`.

#### Quarkus

1. Search `dependencies` for `io.quarkus:quarkus-core` or `io.quarkus.platform:quarkus-bom`.
2. Compare version against `desired_state.frameworks.quarkus.minimum_version`.

#### Micronaut

1. Search `dependencies` for `io.micronaut:micronaut-core` or the Micronaut BOM.
2. Compare version against `desired_state.frameworks.micronaut.minimum_version`.

#### Django

1. Search `dependencies` for `django` (case-insensitive) in pip/poetry ecosystems.
2. Compare version against `desired_state.frameworks.django.minimum_version`.

#### Flask

1. Search `dependencies` for `flask` in pip/poetry ecosystems.
2. Compare version against `desired_state.frameworks.flask.minimum_version`.

### Severity Assignment for Framework Issues

- `critical` — framework version in `deprecated_versions` list.
- `high` — framework version below `minimum_version`.
- `medium` — framework version above minimum but not in `allowed_versions` (if provided).
- `info` — compliant.

### Build Tool Detection

#### Maven

1. Check for `mvnw` wrapper file. If present, read `.mvn/wrapper/maven-wrapper.properties` for `distributionUrl` to extract version.
2. Check `pom.xml` for `<maven.version>` property or `maven-enforcer-plugin` constraints.
3. Compare against `desired_state.build_tools.maven.minimum_version`.

#### Gradle

1. Check `gradle/wrapper/gradle-wrapper.properties` for `distributionUrl` to extract Gradle version.
2. Compare against `desired_state.build_tools.gradle.minimum_version`.

#### npm

1. Read `package.json` `engines.npm` field.
2. Check `.npmrc` for engine constraints.
3. Compare against `desired_state.build_tools.npm.minimum_version`.

### Platform Configuration Checks

1. **Required files**: For each path in `desired_state.platform.required_configurations`, verify that the file or directory exists in the repository. Emit a `PlatformDriftIssue` with severity `high` for any missing required configuration.
2. **Disallowed tools**: Check build files and CI/CD configurations for references to tools listed in `desired_state.platform.disallowed_tools`. Emit a `PlatformDriftIssue` with severity `high` for any detected disallowed tool.
3. **CI/CD required checks**: Parse CI/CD pipeline files and verify that jobs or steps matching the names in `desired_state.platform.ci_cd.required_checks` are defined. Emit a `PlatformDriftIssue` with severity `medium` for each missing required check.
4. **CI/CD provider**: Detect which CI/CD provider is in use and verify it is in `desired_state.platform.ci_cd.allowed_providers`. Emit a `PlatformDriftIssue` with severity `high` if an unapproved provider is detected.

## Constraints

- Framework version detection should work with both direct version declarations and BOM/parent-managed versions.
- When a version cannot be resolved (managed externally and not present in local files), emit an `info`-severity issue noting the version is unresolved.
- Do not make network calls.

## Example Output

```json
{
  "framework_issues": [
    {
      "framework": "spring_boot",
      "detected_version": "2.7.18",
      "minimum_allowed_version": "3.1.0",
      "source_file": "pom.xml",
      "severity": "critical",
      "message": "Spring Boot 2.7.18 is in the deprecated 2.x line. Upgrade to Spring Boot 3.1.x or later."
    }
  ],
  "build_tool_issues": [
    {
      "tool": "maven",
      "detected_version": "3.8.6",
      "minimum_allowed_version": "3.9.0",
      "source_file": ".mvn/wrapper/maven-wrapper.properties",
      "severity": "high",
      "message": "Maven 3.8.6 is below the minimum allowed version of 3.9.0. Update the Maven wrapper to 3.9.x."
    }
  ],
  "platform_drift_issues": [
    {
      "check": "required_configuration",
      "expected": "Dockerfile",
      "actual": null,
      "severity": "high",
      "message": "Required configuration file 'Dockerfile' is missing from the repository."
    },
    {
      "check": "ci_cd_required_check",
      "expected": "security-scan",
      "actual": null,
      "severity": "medium",
      "message": "Required CI/CD check 'security-scan' is not defined in any pipeline configuration."
    }
  ]
}
```
