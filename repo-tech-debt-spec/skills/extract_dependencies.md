# Skill: Extract Dependencies

## Description

Parse build files and dependency manifests to produce a normalized, complete list of all direct and transitive dependencies declared in the repository.

## Inputs

| Input | Type | Required | Description |
|-------|------|----------|-------------|
| `build_files` | list[string] | Yes | Paths to build files identified by the `scan_repository` skill. |
| `repository_path` | string | Yes | Absolute or relative path to the root of the repository. |

## Outputs

| Output | Type | Description |
|--------|------|-------------|
| `dependencies` | list[Dependency] | Normalized list of all dependencies extracted from all build files. |

### Dependency Object Schema

```json
{
  "name": "string",
  "group": "string | null",
  "version": "string | null",
  "scope": "string | null",
  "source_file": "string",
  "ecosystem": "string",
  "is_direct": "boolean"
}
```

| Field | Description |
|-------|-------------|
| `name` | Artifact or package name (e.g., `spring-boot-starter-web`, `express`). |
| `group` | Group or organization identifier, if applicable (e.g., `org.springframework.boot`). Null for ecosystems that don't use groups. |
| `version` | Declared version string. `null` if version is managed by a BOM or parent POM. |
| `scope` | Dependency scope or configuration (e.g., `compile`, `test`, `runtime`, `devDependencies`). |
| `source_file` | Path to the build file this dependency was extracted from. |
| `ecosystem` | Package ecosystem: `maven`, `gradle`, `npm`, `pip`, `pipenv`, `poetry`. |
| `is_direct` | `true` if declared directly in the build file; `false` if inferred as transitive (when lock files are available). |

## Behavior

### Maven (`pom.xml`)

1. Parse the XML structure of each `pom.xml`.
2. Extract all `<dependency>` entries from `<dependencies>` and `<dependencyManagement>`.
3. Capture `groupId`, `artifactId`, `version`, and `scope`.
4. Resolve `${property}` version placeholders using `<properties>` defined in the same POM.
5. If a parent POM is referenced locally, attempt to resolve inherited properties and managed versions from it.

### Gradle (`build.gradle` / `build.gradle.kts`)

1. Parse dependency declarations from `dependencies { }` blocks.
2. Extract `group`, `name`, and `version` from string and map-style declarations.
3. Capture configuration names (e.g., `implementation`, `testImplementation`, `api`).
4. If a `gradle.lockfile` is present, include locked transitive dependencies and mark them as `is_direct: false`.

### npm (`package.json`)

1. Parse `dependencies`, `devDependencies`, `peerDependencies`, and `optionalDependencies`.
2. Extract package name and version constraint.
3. If `package-lock.json` or `yarn.lock` is present, resolve and include locked transitive dependencies.

### Python (`requirements.txt`, `Pipfile`, `pyproject.toml`)

1. For `requirements.txt`: parse each line, extracting package name and version specifier.
2. For `Pipfile`: parse `[packages]` and `[dev-packages]` sections.
3. For `pyproject.toml`: extract `[tool.poetry.dependencies]` or `[project.dependencies]`.
4. If `Pipfile.lock` or `poetry.lock` is present, include locked transitive dependencies.

## Constraints

- Normalize all package names to lowercase.
- Strip leading/trailing whitespace from version strings.
- If a version cannot be resolved (e.g., version managed externally), set `version` to `null` and log a warning.
- Do not attempt network calls to resolve versions; work only with files present in the repository.

## Example Output

```json
{
  "dependencies": [
    {
      "name": "spring-boot-starter-web",
      "group": "org.springframework.boot",
      "version": "2.7.18",
      "scope": "compile",
      "source_file": "pom.xml",
      "ecosystem": "maven",
      "is_direct": true
    },
    {
      "name": "jackson-databind",
      "group": "com.fasterxml.jackson.core",
      "version": "2.13.4",
      "scope": "compile",
      "source_file": "pom.xml",
      "ecosystem": "maven",
      "is_direct": true
    },
    {
      "name": "express",
      "group": null,
      "version": "4.18.2",
      "scope": "dependencies",
      "source_file": "package.json",
      "ecosystem": "npm",
      "is_direct": true
    }
  ]
}
```
