# Skill: Scan Repository

## Description

Scan the target repository to produce a structural inventory of all relevant artifacts that will be used by subsequent analysis skills.

## Inputs

| Input | Type | Required | Description |
|-------|------|----------|-------------|
| `repository_path` | string | Yes | Absolute or relative path to the root of the repository being analyzed. |

## Outputs

| Output | Type | Description |
|--------|------|-------------|
| `build_files` | list[string] | Paths to detected build files (e.g., `pom.xml`, `build.gradle`, `build.gradle.kts`, `package.json`, `requirements.txt`, `pyproject.toml`). |
| `dockerfiles` | list[string] | Paths to all Dockerfiles found in the repository. |
| `ci_cd_files` | list[string] | Paths to CI/CD pipeline configuration files (e.g., `.github/workflows/*.yml`, `Jenkinsfile`, `azure-pipelines.yml`). |
| `runtime_configs` | list[string] | Paths to runtime environment configuration files (e.g., `.nvmrc`, `.java-version`, `.tool-versions`, `runtime.txt`). |
| `source_structure` | object | Summary of detected source directories and their primary programming language. |
| `desired_state_file` | string | Path to the desired state specification file (`specs/desired_state.yml` or `specs/platform.yml`). |

## Behavior

1. Recursively scan the `repository_path` directory.
2. Identify and collect all build files by matching against known filenames:
   - Java: `pom.xml`, `build.gradle`, `build.gradle.kts`
   - JavaScript/TypeScript: `package.json`
   - Python: `requirements.txt`, `Pipfile`, `pyproject.toml`, `setup.py`, `setup.cfg`
3. Identify all files named `Dockerfile` or matching `Dockerfile.*`.
4. Identify CI/CD configuration files:
   - GitHub Actions: `.github/workflows/*.yml`
   - Jenkins: `Jenkinsfile`
   - Azure DevOps: `azure-pipelines.yml`
5. Identify runtime pinning files: `.nvmrc`, `.java-version`, `.tool-versions`, `runtime.txt`.
6. Detect the primary language and framework of each top-level source module based on file extensions and build file presence.
7. Locate the desired state specification file at `specs/desired_state.yml` or `specs/platform.yml`.
8. Return all collected paths and the source structure summary as structured output.

## Constraints

- Do not read file contents during this scan; only collect file paths and metadata.
- Ignore files in `.git/`, `node_modules/`, `vendor/`, `.gradle/`, and other dependency cache directories.
- If no desired state file is found, emit a warning but continue scanning; downstream skills will fail gracefully without it.

## Example Output

```json
{
  "build_files": ["pom.xml", "service-b/pom.xml"],
  "dockerfiles": ["Dockerfile", "service-b/Dockerfile"],
  "ci_cd_files": [".github/workflows/ci.yml", ".github/workflows/release.yml"],
  "runtime_configs": [".java-version"],
  "source_structure": {
    "root": "java",
    "service-b": "java"
  },
  "desired_state_file": "specs/desired_state.yml"
}
```
