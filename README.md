# ci-workflows

Centralized reusable GitHub Actions workflows and composite actions for all gatewaynet projects.

**All pipeline stages run on the shared self-hosted ARC runners on k8s-xeon by default — no billed GitHub-hosted minutes.**

Documentation:
- [docs/000_INTEGRATION_GUIDE.md](docs/000_INTEGRATION_GUIDE.md) — how to integrate the shared pipeline into a project
- [docs/001_SELF_HOSTED_RUNNERS.md](docs/001_SELF_HOSTED_RUNNERS.md) — shared runners on k8s-xeon: labels, scoping, adding coverage, troubleshooting
- [docs/002_CI_FAILOVER.md](docs/002_CI_FAILOVER.md) — automatic failover to GitHub-hosted runners when k8s-xeon is offline

## Quick Start

Add this to your project's `.github/workflows/ci-cd.yml`:

```yaml
name: CI/CD
on:
  pull_request:
    types: [opened, synchronize, reopened, closed, labeled]
    branches: [develop, master]
  push:
    branches: [develop, master]
  issue_comment:
    types: [created]
permissions:
  contents: write
  pull-requests: write
  issues: write

jobs:
  pipeline:
    uses: gatewaynet/ci-workflows/.github/workflows/node-ci-cd.yml@master
    with:
      node-version: "20"
      image-name: "my-service"
      argocd-app-name-develop: "dev-my-service"
      argocd-app-name-production: "my-service"
      argocd-resource-name: "my-service-deployment"
      run-lint: true
      run-tests: true
    secrets: inherit
```

## Repository Structure

```
ci-workflows/
├── docs/                       # Integration guide, runner docs, HA runbook
├── .github/workflows/          # Reusable workflows (workflow_call)
│   ├── node-ci-cd.yml          # Node.js CI/CD pipeline
│   └── pr-cleanup.yml          # Standalone PR image cleanup
├── actions/                    # Composite actions
│   ├── do-registry-login/      # DigitalOcean registry auth
│   ├── docker-build-push/      # Docker buildx + multi-tag push
│   ├── version-tag/            # Auto-increment version tags
│   ├── argocd-restart/         # ArgoCD deployment restart
│   ├── pr-comment/             # PR comment upsert
│   └── change-detection/       # Monorepo service change detection
└── runners/                    # ARC K8s runner infrastructure
    ├── namespace.yaml
    ├── arc-controller/         # ARC controller Helm values
    └── runner-scale-sets/      # Runner scale set configs
        ├── light-runner/       # 2 CPU, 4Gi - lint, small builds
        └── heavy-runner/       # 4 CPU, 8Gi - E2E, race tests, multi-image builds
```

## Workflows

### `node-ci-cd.yml`

Full CI/CD pipeline for Node.js projects. Handles the entire lifecycle:

| Stage | Trigger | What it does |
|-------|---------|--------------|
| **Check** | PR open/sync, push | Lint, typecheck, build, test (configurable) — always runs |
| **PR Preview** | `BUILD_PREVIEW` label or comment | Builds `pr-N` image, comments PR |
| **Develop Build** | Push to develop | Auto-increments `dev-vX.Y.Z`, pushes + `dev-latest` |
| **Release Verify** | PR to master | Build-only verification, comments PR with release info |
| **Production Build** | Push to master | Extracts version from merge commit, pushes + `production-latest` |
| **ArgoCD Restart** | After each build | Restarts deployment (if configured) |
| **PR Cleanup** | PR closed | Deletes preview image from registry |

### Skip Develop Build with `[NOBUILD]`

Add `[NOBUILD]` to a commit message on the develop branch to skip the Docker
build stage. The `check` job (lint, typecheck, test) still runs, but no image
will be built, tagged, or pushed.

**Scope:** Only applies to develop builds. PR previews, release verify, and
production builds are not affected.

Use this for changes that don't require a new image:
- Documentation updates (`README.md`, `docs/`)
- Postman collections / API specs
- Comments, formatting, non-code changes

```bash
# Direct commit to develop
git commit -m "docs: update API examples [NOBUILD]"
git push origin develop

# Via squash-merged PR — put [NOBUILD] in the PR title
# (squash commit uses PR title as message)
```

**Key inputs:**

| Input | Default | Description |
|-------|---------|-------------|
| `node-version` | _(required)_ | Node.js version |
| `image-name` | _(required)_ | Docker image name |
| `run-lint` | `true` | Run linting |
| `run-tests` | `false` | Run tests |
| `run-build` | `false` | Run build check |
| `run-typecheck` | `false` | Run type checking |
| `run-audit` | `false` | Run security audit |
| `check-runner` | `self-hosted-light` | Runner for check jobs |
| `build-runner` | `self-hosted-heavy` | Runner for Docker builds |
| `utility-runner` | `self-hosted-light` | Runner for tags, version sync, ArgoCD restarts, cleanup |
| `dockerfile` | `Dockerfile` | Dockerfile path |
| `docker-build-args` | `""` | Extra build args (newline-separated) |

**Required secrets:** `DO_API_TOKEN`. Optional: `ARGOCD_AUTH_TOKEN`, `ARGOCD_SERVER`.

## Composite Actions

Use individually in custom workflows:

```yaml
- uses: gatewaynet/ci-workflows/actions/do-registry-login@master
  with:
    do-api-token: ${{ secrets.DO_API_TOKEN }}

- uses: gatewaynet/ci-workflows/actions/docker-build-push@master
  with:
    registry: registry.digitalocean.com/gatewaynet-service-repo
    image-name: my-service
    version-tag: dev-v1.0.5
    extra-tags: dev-latest
    commit-sha: ${{ github.sha }}
    build-number: ${{ github.run_number }}

- uses: gatewaynet/ci-workflows/actions/version-tag@master
  with:
    environment: develop  # develop | production | preview

- uses: gatewaynet/ci-workflows/actions/argocd-restart@master
  with:
    app-name: dev-my-service
    resource-name: my-service-deployment
    argocd-server: ${{ secrets.ARGOCD_SERVER }}
    argocd-auth-token: ${{ secrets.ARGOCD_AUTH_TOKEN }}
```

## Self-Hosted Runners

Two runner scale sets via ARC (Actions Runner Controller) on the K8s cluster:

| Runner | Label | Resources | Use for |
|--------|-------|-----------|---------|
| Light | `self-hosted-light` | 2 CPU, 4Gi | Checks, tags, ArgoCD restarts, cleanup |
| Heavy | `self-hosted-heavy` | 4 CPU, 8Gi + DinD | Docker builds, E2E, Go race tests |

These are the defaults for every stage. Your repo must be covered by a
matching runner scale set (org-wide or repo-scoped) or jobs will queue
forever — see [docs/001_SELF_HOSTED_RUNNERS.md](docs/001_SELF_HOSTED_RUNNERS.md).

Opt back into GitHub-hosted (billed) per repo if ever needed:
```yaml
with:
  check-runner: "ubuntu-latest"
```

Disaster failover: the watchdog sets the repo variable `CI_RUNNER_OVERRIDE` on every repo to redirect
all stages at once ([docs/002_CI_FAILOVER.md](docs/002_CI_FAILOVER.md)).

## Required Repository Setup

Each calling repository needs:

1. **Secrets:** `DO_API_TOKEN`, `ARGOCD_AUTH_TOKEN` (per environment)
2. **Variables:** (optional, falls back to inputs)
3. **Environments:** `develop`, `production` in GitHub Settings
4. **Labels:** Create `BUILD_PREVIEW` label for PR preview builds
