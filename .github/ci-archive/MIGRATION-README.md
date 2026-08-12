# Jenkins to GitHub Actions Migration Report

## Summary

Migrated the repository's Jenkins pipeline definitions to a consolidated GitHub Actions workflow at `.github/workflows/jenkins-migration.yml` and archived the original Jenkins files under `.github/ci-archive/`.

## Source Jenkins files

| Original file | Archived file | Pipeline type | Notes |
| --- | --- | --- | --- |
| `Jenkinsfile` | `.github/ci-archive/Jenkinsfile` | Declarative | Initializes Maven-derived environment values and prints runtime context. The Jenkins `Build` stage was disabled with `when { expression { false } }`; no active build job was migrated for that disabled stage. |
| `parameterstriggers/Jenkinsfile` | `.github/ci-archive/parameterstriggers/Jenkinsfile` | Declarative | Migrated parameters, cron-style triggers, concurrency behavior, timeout, checkout, and simple build commands. |
| `toolsagents/Jenkinsfile` | `.github/ci-archive/toolsagents/Jenkinsfile` | Declarative | Migrated Linux/JDK/Maven build, test, Surefire/Jacoco artifact publication, and Sonar quality steps. |

No Jenkins shared libraries, `vars/` functions, `@Library` annotations, or `library(...)` calls were present, so no shared-library expansion was required.

## Workflow created

Created `.github/workflows/jenkins-migration.yml` with these jobs:

- `root-init`: migrates the active `Init` stage from the root Jenkinsfile.
- `parameters-triggers`: migrates parameterized setup/build behavior and Jenkins trigger intent.
- `tools-agents-build`: migrates the Maven/JDK build stage.
- `tools-agents-test`: migrates the Maven test stage and publishes Surefire/Jacoco outputs as artifacts.
- `tools-agents-quality`: migrates the Sonar quality stage.

## Trigger mapping

| Jenkins | GitHub Actions |
| --- | --- |
| Manual parameterized build | `workflow_dispatch` inputs: `flag`, `SOME_STRING`, `ENVIRONMENT` |
| `cron('@daily')` | `schedule` with `0 0 * * *` |
| `pollSCM('H/5 * * * *')` | `schedule` with `*/5 * * * *` and `push` |
| `disableConcurrentBuilds()` | Workflow-level `concurrency` with `cancel-in-progress: false` |
| `timeout(time: 10, unit: 'MINUTES')` | Job-level `timeout-minutes: 10` |
| `buildDiscarder(logRotator(numToKeepStr:'5'))` | Artifact `retention-days: 5` where artifacts are uploaded |

## Tools and agents mapping

| Jenkins | GitHub Actions |
| --- | --- |
| `agent any` / `label 'linux'` | `runs-on: ubuntu-latest` |
| `label 'sonar'` | `runs-on: ubuntu-latest` with Sonar token gating |
| Maven `apache-maven-3.6.3` | Runner Maven installation with `actions/setup-java` Maven cache |
| JDK `jdk-11` | `actions/setup-java` with Temurin Java 11 |
| `archive`, `stash`, test result, and coverage publication | `actions/upload-artifact` |

The repository currently does not include `pom.xml` or `mvnw`. Maven-dependent migrated steps are therefore guarded so the workflow remains valid for the current repository contents while still running the original Maven commands when a Maven project is present.

## Secrets and variables

| Jenkins credential | GitHub Actions secret | Usage |
| --- | --- | --- |
| `sonar-token` | `SONAR_TOKEN` | Used by `tools-agents-quality` for `mvn sonar:sonar -Dsonar.login="${SONAR_TOKEN}"` |

No credential values were present in the Jenkins files, and no secret values were migrated into the repository.

## Action security

Marketplace actions are pinned to immutable commit SHAs:

| Action | Version source | Commit SHA |
| --- | --- | --- |
| `actions/checkout` | v4 | `11d5960a326750d5838078e36cf38b85af677262` |
| `actions/setup-java` | v4 | `cf277c60eb25467037889841efdb72551f06f6c3` |
| `actions/upload-artifact` | v4 | `ea165f8d65b6e75b540449e92b4886f43607fa02` |

The workflow also limits the default `GITHUB_TOKEN` to read-only repository contents access with `permissions: contents: read`.

## Validation

- Run `actionlint .github/workflows/jenkins-migration.yml` to validate workflow syntax.
- Run `git diff --check` to validate whitespace.
- Maven build/test validation is skipped for the current repository state because no Maven project files are present.

Migration complete. MIGRATION-README.md created and Pull Request updated/created with migration report.
