# FO-AI automation

Reusable GitHub Actions workflows. Apps supply their own test commands, Dockerfiles,
deployment scripts, and settings.

## Naming

| Location | File | Purpose |
| --- | --- | --- |
| App repo | `.github/workflows/ci.yml` → **CI** | Runs separate **Frontend** and **Backend** jobs, then optional Docker builds. |
| App repo | `.github/workflows/cd.yml` → **CD** | Calls Azure CD after successful main CI. |
| App repo | `scripts/ci-frontend.sh`, `scripts/ci-backend.sh` | App-owned check commands. |
| This repo | [reusable-ci.yml](.github/workflows/reusable-ci.yml) | Installs runtimes and runs the supplied checks/builds. |
| This repo | [reusable-azure-cd.yml](.github/workflows/reusable-azure-cd.yml) | Selects the environment, validates the commit, signs into Azure, and runs the app's deployment script. |
| This repo | [.github/workflows/ci.yml](.github/workflows/ci.yml) | Tests these reusable workflows. |

Use a tested full commit SHA in callers. The new filenames are the **v2 interface**;
`v1` and `v1.0.0` retain the old files. Upgrade the filename and commit pin together.
Keep existing pins until the new caller passes its checks. An Actions sidebar shows
the app's workflow name, not the reusable workflow's name.

## CI caller

Create `.github/workflows/ci.yml` in the app. Replace `RELEASE_SHA` with the tested
v2 release commit, and supply your paths, installs, and scripts:

```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]
permissions:
  contents: read
jobs:
  ci:
    uses: FO-AI/automation/.github/workflows/reusable-ci.yml@RELEASE_SHA
    with:
      checks: >-
        [
          {"name":"Frontend","directory":".","node-version":"22",
           "install":"npm --prefix frontend ci","run":"bash scripts/ci-frontend.sh"},
          {"name":"Backend","directory":".","python-version":"3.12","uv-version":"0.12.15",
           "install":"uv sync --project backend --locked --extra dev","run":"bash scripts/ci-backend.sh"}
        ]
```

Each matrix entry creates a separate parallel job and check result. Both checks
run even when one fails. Optional Docker builds wait for **all** checks to pass.

| Input | Value |
| --- | --- |
| `checks` (required) | Nonempty JSON array; each item needs `name`, `directory`, `install`, and `run`. |
| `containers` (optional) | JSON array of `{name, context, file, build-args?}`. Builds only; never publishes. |

Optional check fields: `node-version`, `python-version`, `pnpm-version`, `uv-version`,
and nonsecret `env`. Use locked dependency installs. CI reads the calling app's source;
do not pass application secrets or credentials in checks or build arguments.

## CD caller

Create `.github/workflows/cd.yml` in the app:

```yaml
name: CD
on:
  workflow_run:
    workflows: [CI]
    types: [completed]
    branches: [main]
  workflow_dispatch:
permissions: {}
jobs:
  deploy:
    uses: FO-AI/automation/.github/workflows/reusable-azure-cd.yml@RELEASE_SHA
    permissions:
      contents: read
      id-token: write
    with:
      environment: dev
      allow-manual: true
      deploy-command: bash scripts/deploy.sh
    secrets: inherit
```

Before enabling CD, configure the app's GitHub environment, branch restrictions,
Azure OIDC identity/roles, and deployment script. `deploy.sh` is app-owned; a Python
app may use `python scripts/deploy.py`. The script owns publishing, deployment,
health checks, recovery, and temporary secret-file cleanup.

Set `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, and `AZURE_SUBSCRIPTION_ID` as caller
variables or secrets. Secrets take precedence; selected environment secrets
override passed secrets of the same name. Optional inputs include runtime versions
and nonsecret `deployment-vars`; see the [workflow inputs](.github/workflows/reusable-azure-cd.yml).

The script receives `DEPLOY_SHA`, `IMAGE_TAG`, read-only `GH_TOKEN`, and
`DEPLOYMENT_VARS_JSON` resolved inside the selected environment. Read only the keys
your app needs. Optional `API_ENV_FILE` and `WEB_ENV_FILE` secrets are exposed to
the deployment command when configured.

The wrapper accepts successful same-repo main-push CI runs and direct main pushes.
Manual deployment is opt-in and accepts current main only; it does not require a
prior successful CI run. Keep historical-image recovery in the app.

Deployments run one at a time per repo/environment without cancelling an active run.
Stale commits fail before Azure login. Recheck main in the app script immediately
before updating Azure because main can advance while an image builds.

## Maintenance

Run `actionlint .github/workflows/*.yml`, the included CI self-tests, and an app
pilot before publishing a release. Keep required check names aligned with the
caller: normally `ci / Frontend`, `ci / Backend`, and its Docker build checks.
Apps with an aggregate gate can require that gate instead.

The v2 rename changes paths and display names only; inputs, permissions, and
deployment guards retain their existing behavior. Existing SHA pins and v1 tags
continue to resolve the original workflows. Callers using the old paths on `main`
must migrate. Reusable workflows should always use a release or commit pin.

Public callers require a public shared repo. Keep app configuration in app repos
and credentials in GitHub secrets or Azure. External actions are SHA-pinned.
