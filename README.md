# FO-AI automation

Reusable GitHub Actions workflows. Apps supply their own check, publish, and deployment
scripts and settings, following the repository script contract below.

## Naming

| Location | File | Purpose |
| --- | --- | --- |
| App repo | `.github/workflows/ci.yml` → **CI** | One `checks` call to the reusable CI (N named checks plus optional Docker builds), a `verify` gate, and on `main` a `publish` job. |
| App repo | `.github/workflows/cd.yml` → **CD** | Calls the reusable Azure CD after successful main CI. |
| App repo | `scripts/ci.sh`, `scripts/publish.sh`, `scripts/cd.sh` | The three app-owned scripts; see [Repository script contract](#repository-script-contract). |
| This repo | [reusable-ci.yml](.github/workflows/reusable-ci.yml) | Installs runtimes and runs the supplied checks/builds. |
| This repo | [reusable-azure-cd.yml](.github/workflows/reusable-azure-cd.yml) | Selects the environment, validates the commit, signs into Azure, and runs the app's deployment script. |
| This repo | [.github/workflows/ci.yml](.github/workflows/ci.yml) | Tests CI runtimes, check execution, and Docker builds. Azure CD needs an app deployment pilot. |

Use a tested full commit SHA in callers. The new filenames are the **v2 interface**;
`v1` and `v1.0.0` retain the old files. Upgrade the filename and commit pin together.
Keep existing pins until the new caller passes its checks. An Actions sidebar shows
the app's workflow name, not the reusable workflow's name.

## Repository script contract

Every FO-AI app repo has exactly three scripts, each with one job and one input surface,
and two caller workflows whose shape is identical across repos. A maintainer moving
between repos finds the same three files doing the same three jobs; a new repo copies the
two YAML files below and writes the three scripts.

| Script | Does | Must not | Receives |
| --- | --- | --- | --- |
| `scripts/ci.sh <check>` | Runs one named check (`backend`, `frontend`, …). Sets its own test-only env so it behaves identically on a laptop. Exits nonzero on failure. | Touch any cloud service. Read secrets. | Nothing beyond the check name; dependencies are installed by the caller. |
| `scripts/publish.sh` | Builds every image for `$IMAGE_TAG`, pushes each to ACR as `<repo>-<service>:$IMAGE_TAG`, and smoke-tests what it pushed (at minimum, proves both tags resolve to digests). | Deploy. Tag anything `latest`. | `ACR_NAME`, `IMAGE_TAG` (a full commit SHA), an `az` login with push rights, plus any nonsecret build args as repository variables. |
| `scripts/cd.sh` | Runs inside `reusable-azure-cd.yml`. Resolves the digests published for `$IMAGE_TAG` (`az acr repository show --image <repo>:$IMAGE_TAG --query digest`), promotes **by digest**, rechecks main immediately before its first change to Azure, verifies. | Build. Accept a digest from its environment. | From the wrapper: `DEPLOY_SHA`, `IMAGE_TAG`, `GH_TOKEN`, `DEPLOYMENT_VARS_JSON`, `API_ENV_FILE`, `WEB_ENV_FILE`. |

Publishing is a local job in each repo rather than part of the reusable workflows, so the
Azure CD wrapper never needs image-build machinery and the deploy job needs no `resolve`
step: `cd.sh` looks the digests up itself after the wrapper has signed in. Splitting
publish from deploy also means a deploy never rebuilds, so what CI smoke-tested is what
runs.

Two OIDC federated credentials per repo: `publish` runs with no environment and emits the
`:ref:refs/heads/main` subject; `deploy` runs in the `dev` environment and emits
`:environment:dev`. Because `publish` cannot read environment-scoped values, its Azure and
ACR identifiers and any build-time arguments are **repository** variables.

Reference implementations: `FO-AI/benny` (buildx + registry cache in `publish.sh`, a local
`e2e` exception job, and a separate historical-recovery job in `cd.yml`) and `FO-AI/nimbus`
(`az acr build` in `publish.sh`, no exceptions).

## CI caller

Create `.github/workflows/ci.yml` in the app. Replace `RELEASE_SHA` with the tested
v2 release commit, and supply your paths and installs. Job names surface as
`checks / backend`, `checks / frontend`, `checks / Build api`, and so on; `verify` is
the single status check to require.

```yaml
name: CI                          # cd.yml triggers on this name
on:
  pull_request:
  push:
    branches: [main]
concurrency:
  group: ci-${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true
permissions:
  contents: read
jobs:
  checks:                         # reusable, one call, N entries
    uses: FO-AI/automation/.github/workflows/reusable-ci.yml@RELEASE_SHA
    with:
      checks: >-
        [{"name":"backend",  "directory":".", "python-version":"3.11",
          "install":"...", "run":"bash scripts/ci.sh backend"},
         {"name":"frontend", "directory":".", "node-version":"22",
          "install":"...", "run":"bash scripts/ci.sh frontend"}]
      containers: >-              # PR-time "does it still build" check; never pushes
        [{"name":"api","context":".","file":"api/Dockerfile"},
         {"name":"web","context":"web","file":"web/Dockerfile"}]

  # Optional, documented exception: a repo-specific local job that needs runner
  # machinery the reusable call cannot provide (e.g. uploading failure artifacts).

  verify:                         # the single required status check
    if: always()
    needs: [checks]               # plus any local exception jobs
    runs-on: ubuntu-latest
    steps:
      - run: test "${{ needs.checks.result }}" = success

  publish:                        # push to main only; the only job with cloud access
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    needs: verify
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write             # no `environment:`, so repo-level variables only
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v2
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      - run: az acr login --name "$ACR_NAME"       # only if the script uses docker push
        env: { ACR_NAME: "${{ vars.ACR_NAME }}" }
      - run: bash scripts/publish.sh
        env:
          ACR_NAME: ${{ vars.ACR_NAME }}
          IMAGE_TAG: ${{ github.sha }}
```

Each matrix entry creates a separate parallel job and check result. All checks run even
when one fails. Optional Docker builds wait for **all** checks to pass.

| Input | Value |
| --- | --- |
| `checks` (required) | Nonempty JSON array; each item needs `name`, `directory`, `install`, and `run`. |
| `containers` (optional) | JSON array of `{name, context, file, build-args?}`. Builds only; never publishes. |

Optional check fields: `node-version`, `python-version`, `pnpm-version`, `uv-version`,
and nonsecret `env` (prefer setting test-only values inside `ci.sh` so the check runs the
same way on a laptop). Use locked dependency installs. CI reads the calling app's source;
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
permissions:
  contents: read
jobs:
  deploy:
    if: >-
      (github.event_name == 'workflow_dispatch' && github.ref == 'refs/heads/main') ||
      (github.event_name == 'workflow_run' &&
       github.event.workflow_run.conclusion == 'success' &&
       github.event.workflow_run.event == 'push' &&
       github.event.workflow_run.head_branch == 'main' &&
       github.event.workflow_run.head_repository.full_name == github.repository)
    uses: FO-AI/automation/.github/workflows/reusable-azure-cd.yml@RELEASE_SHA
    permissions:
      contents: read
      id-token: write
    with:
      environment: dev
      allow-manual: true
      deploy-command: bash scripts/cd.sh
    secrets: inherit
```

Before enabling CD, configure the app's GitHub environment, branch restrictions,
Azure OIDC identity/roles, and `scripts/cd.sh`. The script owns digest resolution,
configuration, health checks, recovery, and temporary secret-file cleanup. Image
build/push belongs in `scripts/publish.sh`, run by the app's own CI `publish` job.

If your Azure trust policy restricts `job_workflow_ref`, allow the new
`reusable-azure-cd.yml` path and chosen pin before upgrading the caller.

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
before updating Azure because main can advance between CI's publish and the deploy.

## Maintenance

Run `actionlint .github/workflows/*.yml`, the included CI self-tests, and an app
pilot before publishing a release. Require the caller's `verify` gate rather than
individual check names, so adding or renaming a check does not need a branch-rule edit.

The v2 rename changes paths and display names only; inputs, permissions, and
deployment guards retain their existing behavior. Existing SHA pins and v1 tags
continue to resolve the original workflows. Callers using the old paths on `main`
must migrate. Reusable workflows should always use a release or commit pin.

Public callers require a public shared repo. Keep app configuration in app repos
and credentials in GitHub secrets or Azure. External actions are SHA-pinned.
