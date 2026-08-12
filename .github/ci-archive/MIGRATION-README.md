# Jenkins to GitHub Actions migration report

## Source mapping

| Archived Jenkins pipeline | GitHub Actions workflow | Preserved behavior |
| --- | --- | --- |
| `Jenkinsfile` | `.github/workflows/netlify-deploy.yml` | Installs dependencies, builds, then deploys `build` to Netlify production. |
| `stagedisabling/Jenkinsfile` | `.github/workflows/maven-stage-disabling.yml` | Prints Maven/Git metadata and retains the intentionally disabled Maven build step. |
| `whenconditions/Jenkinsfile` | `.github/workflows/branch-conditions.yml` | Runs the common message and preserves the `master`/non-`master` branch conditions and always-false expression. |

## Triggers and credentials

The Jenkinsfiles did not declare triggers, so each workflow is manually started with
`workflow_dispatch`. The Netlify credential `netlify-token` must be created as the
repository secret `NETLIFY_AUTH_TOKEN`; `NETLIFY_SITE_ID` remains
`classy-paletas-f45a67`.

## Notes

No Jenkins shared libraries were used. The Maven workflow requires the repository
content expected by the original pipeline, including `mvnw` and its project POM.
The original disabled build stage remains disabled in GitHub Actions.

## Validation

The workflows were checked with `yamllint`. `actionlint` was not installed in the
execution environment.
