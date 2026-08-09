# Integrating the Shared CI/CD Workflows

How any gatewaynet project plugs into the reusable pipelines in this repo.
For runner infrastructure (labels, sizing, adding coverage for a new repo) see
[001_SELF_HOSTED_RUNNERS.md](001_SELF_HOSTED_RUNNERS.md).

## 1. Minimal caller workflow

Create `.github/workflows/ci-cd.yml` in your repo:

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
      run-lint: true
      run-tests: true
    secrets: inherit
```

That's the whole pipeline: check → PR preview → develop build → release
verify → production build → ArgoCD restart → PR cleanup.

## 2. One-time repository setup

| Item | Where | Value |
|---|---|---|
| Secret `DO_API_TOKEN` | Repo secrets | DigitalOcean registry token (required) |
| Environments | Settings → Environments | `develop`, `production` (+ `test` if using test restarts) |
| Secret `ARGOCD_AUTH_TOKEN` | Per environment | ArgoCD API token (only if restarts enabled) |
| Var `ARGOCD_SERVER` | Per environment | `argocd.clanroot.dev` (test/develop) / `argocd.clanroot.net` (production) |
| Var `ARGOCD_APP_NAME` | Per environment | ArgoCD application name (no env prefix on prod) |
| Var `ARGOCD_DEPLOYMENTS` | Per environment | Deployment name(s), comma-separated, or `all` |
| Label `BUILD_PREVIEW` | Repo labels | Enables PR preview builds (label mode) |

Repos are provisioned via Terragrunt in `infra.env/github/` — new repos should
get these secrets/vars/environments there, not by hand.

## 3. Pipeline stages and triggers

| Stage | Trigger | Runner (default) |
|---|---|---|
| Check (lint/typecheck/build/test) | Every PR sync + push | `self-hosted-light` |
| PR Preview Build | `BUILD_PREVIEW` label or PR comment (`pr-preview-mode: label`), or every PR to develop (`auto`) | `self-hosted-heavy` |
| Develop Build + `dev-vX.Y.Z` tag | Push to `develop` | `self-hosted-heavy` / `self-hosted-light` |
| Release Verify (build only, no push) | PR targeting `master` | `self-hosted-heavy` |
| Production Build + `vX.Y.Z` tag | Push to `master` (merge commit must reference `release: vX.Y.Z`) | `self-hosted-heavy` / `self-hosted-light` |
| Version sync to develop | After production release | `self-hosted-light` |
| ArgoCD restarts (test/develop/production) | After the matching build, per `enable-*-restart` inputs | `self-hosted-light` |
| PR image cleanup | PR closed | `self-hosted-light` |

**All stages run on the shared self-hosted runners by default** — no
GitHub-hosted minutes are consumed. Override per repo only if you have a
reason:

```yaml
with:
  check-runner: "ubuntu-latest"        # opt back into GitHub-hosted (billed)
  build-runner: "self-hosted-heavy"
  utility-runner: "self-hosted-light"
```

A repo variable `CI_RUNNER_OVERRIDE` (reconciled across all repos by the failover watchdog) overrides **all** stages at
once (disaster failover — see
[002_CI_FAILOVER.md](002_CI_FAILOVER.md)). Leave it unset
in normal operation.

## 4. Commonly used inputs

| Input | Default | Notes |
|---|---|---|
| `node-version` | _(required)_ | Passed to `actions/setup-node` |
| `image-name` | _(required)_ | Image name in the DO registry |
| `registry` | `registry.digitalocean.com/gatewaynet-service-repo` | |
| `run-lint` / `run-tests` / `run-build` / `run-typecheck` / `run-audit` | `true`/`false`/`false`/`false`/`false` | Each has a matching `*-command` input |
| `install-command` | `npm ci` | Use `pnpm install --frozen-lockfile` etc. |
| `dockerfile` | `Dockerfile` | |
| `docker-build-args` | `""` | Newline-separated `KEY=value` pairs |
| `pr-preview-mode` | `label` | `label` / `auto` / `disabled` |
| `develop-branch` / `production-branch` | `develop` / `master` | |
| `enable-test-restart` | `true` | Uses the `test` GitHub environment |
| `enable-develop-restart` / `enable-production-restart` | `false` | |
| `check-runner` / `build-runner` / `utility-runner` | `self-hosted-light` / `self-hosted-heavy` / `self-hosted-light` | |

## 5. Skipping a develop build

Add `[NOBUILD]` to the commit message (or PR title for squash merges) on
`develop`. Checks still run; no image is built. Develop builds only.

## 6. Multi-image / monorepo projects

Use the composite actions directly (`actions/change-detection`,
`actions/docker-build-push`, `actions/version-tag` with `tag-prefix`) in a
custom workflow, and `pr-cleanup.yml` with `image-names` (comma-separated)
for closing PRs. See the README for per-action usage snippets.
