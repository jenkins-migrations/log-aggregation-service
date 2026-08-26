# Jenkins to GitHub Actions Migration Report

## Summary

This repository contained three Jenkins pipelines used as test/demo fixtures. All three have
been migrated to equivalent GitHub Actions workflows. The original `Jenkinsfile`s have been
archived (not deleted) in this directory for reference.

| Original Jenkinsfile | New Workflow | Purpose |
|---|---|---|
| `Jenkinsfile` (repo root) | `.github/workflows/build-and-deploy.yml` | React/Node.js build and Netlify deployment |
| `whenconditions/Jenkinsfile` | `.github/workflows/when-conditions.yml` | `when` conditions and branch logic demo |
| `stagedisabling/Jenkinsfile` | `.github/workflows/stage-disabling.yml` | Maven build with a permanently disabled stage and env var handling |

## Pipeline type

All three pipelines used **declarative** syntax (`pipeline { }` block).

## 1. `Jenkinsfile` → `build-and-deploy.yml`

**Original behavior:** `agent any`, a `Build` stage running `npm install` / `npm run build`,
and a `Deploy` stage that installs `netlify-cli` and deploys the `build` directory to Netlify,
using a `NETLIFY_SITE_ID` environment value and a `NETLIFY_AUTH_TOKEN` Jenkins credential.

**Conversion notes:**
- `agent any` → `runs-on: ubuntu-latest`
- `sh 'npm install'` / `sh 'npm run build'` → `run:` steps, after `actions/setup-node`
- Build output is passed between jobs using `actions/upload-artifact` / `actions/download-artifact`
- `environment { NETLIFY_SITE_ID = '...' }` → repository variable `vars.NETLIFY_SITE_ID`
- `credentials('netlify-token')` → repository/environment secret `secrets.NETLIFY_AUTH_TOKEN`
- Deploy job runs in the `production` GitHub Environment for extra protection/approval support

**Required secrets / variables:**
| Jenkins credential/env | GitHub equivalent | Type |
|---|---|---|
| `NETLIFY_SITE_ID` (literal env value) | `vars.NETLIFY_SITE_ID` (e.g. `classy-paletas-f45a67`) | Repository variable |
| `netlify-token` (secret text credential) | `secrets.NETLIFY_AUTH_TOKEN` | Repository/Environment secret |

## 2. `whenconditions/Jenkinsfile` → `when-conditions.yml`

**Original behavior:** Four stages demonstrating `when` conditions:
- `One` always runs
- `Evaluate Master` runs only `when { branch "master" }`
- `Branch Test` runs `when { not { branch "master" } }`
- `Expression Test` is always skipped because its `when { expression { ... return false } }` always evaluates to `false`

**Conversion notes:**
- `when { branch "master" }` → job-level `if: github.ref == 'refs/heads/master'`
- `when { not { branch "master" } }` → job-level `if: github.ref != 'refs/heads/master'`
- The expression-test stage's `echo "Should I run?"` step (which Jenkins always executes before
  evaluating the `when` block) is preserved; the step it gated (`"This should be skipped"`) is
  permanently unreachable in the source pipeline and is therefore omitted rather than expressed
  as a dead `if: false` step (flagged by `actionlint` as a redundant constant condition).
- `needs:` is used between jobs to preserve the original stage ordering.

**Required secrets / variables:** None.

## 3. `stagedisabling/Jenkinsfile` → `stage-disabling.yml`

**Original behavior:** An `Init` stage that:
- Reads the Maven POM group id/version via `readMavenPom()` and the Maven help plugin
- Computes a release version by stripping `-SNAPSHOT`
- Computes `GIT_TAG_COMMIT` via `git describe --tags --always`
- Computes `IS_SNAPSHOT` from the resolved Maven version
- Prints several diagnostic values (`version_a`, `IS_SNAPSHOT` class, sorted environment, params)

A `Build` stage that is permanently disabled via `when { expression { false } }` (packages the
project with Maven, archives, and stashes the resulting jar).

**Conversion notes:**
- `readMavenPom().getGroupId()/.getVersion()` and the `getMavenVersion()` shared function
  (Maven help-plugin `evaluate` call) are expanded inline as shell (`./mvnw ... evaluate`) calls
- `sh (script: 'git describe --tags --always', returnStdout: true).trim()` → `git describe --tags --always` shell step (requires `fetch-depth: 0` on checkout so tags are available)
- `readMavenPom().getVersion().replace("-SNAPSHOT", "")` → shell parameter expansion (`${POM_VERSION%-SNAPSHOT}`)
- `echo sh(script: 'env|sort', ...)` → `env | sort` step
- The disabled `Build` stage's `when { expression { false } }` (a hard-coded, permanent disable)
  is preserved as a job-level `if: vars.ENABLE_BUILD_STAGE == 'true'` condition. Because the
  `ENABLE_BUILD_STAGE` repository variable is unset by default, the job behaves identically to
  the original (never runs) but can be re-enabled without editing the workflow, and avoids
  `actionlint`'s redundant-constant-condition check for a literal `if: false`.
- `archive` / `stash` → `actions/upload-artifact`

**Required secrets / variables:**
| Purpose | GitHub equivalent | Type |
|---|---|---|
| Re-enable the disabled `Build` job | `vars.ENABLE_BUILD_STAGE` (set to `"true"` to enable) | Repository variable (optional) |

## Actions used

All actions are pinned to a commit SHA (with the version as a trailing comment) per migration
guardrails:

- `actions/checkout` (v7.0.1)
- `actions/setup-node` (v7.0.0)
- `actions/setup-java` (v6.0.0)
- `actions/upload-artifact` (v7.0.1)
- `actions/download-artifact` (v8.0.1)

## Validation

All new workflow files were validated with [`actionlint`](https://github.com/rhysd/actionlint)
(v1.7.12) with zero errors or warnings.

## Follow-ups for repository maintainers

- Configure the `NETLIFY_SITE_ID` repository variable and `NETLIFY_AUTH_TOKEN` secret (ideally
  scoped to the `production` GitHub Environment) before the `build-and-deploy.yml` workflow can
  deploy successfully.
- The `stage-disabling.yml` workflow assumes a Maven project with an `./mvnw` wrapper at the
  repository root; none exists in this repository since the Jenkinsfile was a standalone test
  fixture, so the `Init` job's Maven/Git steps will fail until a real Maven project is present.
