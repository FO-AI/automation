# FO-AI automation

Shared GitHub Actions workflows for FO-AI repos. Each app supplies its checks and deployment script.

Use a verified release or full commit SHA. The examples below use `v1`;
commit pins let each app review shared pipeline upgrades individually.

## CI

Create `.github/workflows/ci.yml` in the app:

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
    uses: FO-AI/automation/.github/workflows/ci.yml@v1
    with:
      checks: >-
        [{"name":"web","directory":"web","node-version":"22",
          "install":"npm ci","run":"npm run lint && npm test && npm run build"}]
```

| Input | Value |
| --- | --- |
| `checks` (required) | Nonempty JSON array; each check needs `name`, `directory`, `install`, and `run`. |
| `containers` (optional) | JSON array of `{name, context, file, build-args?}` objects. Builds after checks pass; never publishes. |

See [CI input definitions](.github/workflows/ci.yml) for optional runtimes and configuration.
Commands can call app scripts. Use locked installs such as `npm ci`,
`pnpm install --frozen-lockfile`, or `uv sync --locked`.

CI reads the calling app's source with no application secrets or Azure access.
Keep `env` and build arguments nonsecret.

## Azure deployment wrapper

Add this job under `jobs` in the same file to deploy after CI passes on `main`:

```yaml
  deploy:
    needs: ci
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    permissions:
      contents: read
      id-token: write
    uses: FO-AI/automation/.github/workflows/deploy-azure.yml@v1
    with:
      environment: dev
      deploy-command: bash scripts/deploy.sh
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      API_ENV_FILE: ${{ secrets.API_ENV_FILE }}
      WEB_ENV_FILE: ${{ secrets.WEB_ENV_FILE }}
```

Before enabling deployment:

1. Add `scripts/deploy.sh` to the app; it is **not included**. The script owns image
   build/push, configuration, health checks, rollback, and temporary secret-file cleanup.
2. Configure the caller's GitHub environment with branch restrictions and available
   review protection. Its secrets override passed secrets with the same names.
3. Set up Azure OIDC trust and roles for **each calling repo/environment**. Verify
   subject claims and any `job_workflow_ref` restrictions.

The three `AZURE_*` IDs can be GitHub variables or secrets. The wrapper uses secrets
first, then variables; omit those secret mappings when using variables. Missing IDs
fail validation before Azure login.

The script receives `DEPLOY_SHA` and `IMAGE_TAG` for the checked-out commit, and
`GH_TOKEN` for read-only GitHub checks. `DEPLOYMENT_VARS_JSON` contains the caller's
GitHub variables resolved inside the selected environment. Read only the keys your
app needs; use this for environment-scoped settings that caller inputs cannot access.
`API_ENV_FILE` and `WEB_ENV_FILE` are optional secrets exposed only to that step;
omit their mappings if unused. See [deployment inputs](.github/workflows/deploy-azure.yml)
for runtimes and nonsecret `deployment-vars`.

Deployments run one at a time per repo/environment without cancelling an active run.
A stale commit is rejected before Azure login; `main` can still advance during deployment.
Keep a final stale-commit check in the app script immediately before changing Azure.
The wrapper also accepts successful `workflow_run` events from same-repo pushes to
`main`. Set `allow-manual: true` to accept `workflow_dispatch` on current `main`;
this does not require a prior CI run. It cannot deploy older commits. Keep existing
historical recovery workflows until migrated and verified.

## Releases and access

Public callers need a public shared repo. Private shared repos require eligible
private callers and Settings → Actions → General → Access configuration.
Keep app configuration in app repos and secrets in GitHub environments.

Protect the default branch and review changes. Run
`actionlint .github/workflows/*.yml` and test in a disposable caller before tagging.
The included `verify.yml` also checks runtimes, matrix options, and Docker builds on
PRs and pushes to this repo's `main`.
Moving `@v1` delivers compatible updates centrally; pin a full commit SHA for
individually reviewed upgrades. External actions are SHA-pinned.
