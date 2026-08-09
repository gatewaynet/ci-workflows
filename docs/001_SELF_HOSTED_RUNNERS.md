# Self-Hosted Runners on k8s-xeon

The shared GitHub Actions runners run on the **k8s-xeon** (test) cluster via
ARC (Actions Runner Controller), namespace `github-runners`. They exist so
that no pipeline stage consumes billed GitHub-hosted minutes.

## Runner classes

| Label (`runs-on`) | Resources | Docker | Use for |
|---|---|---|---|
| `self-hosted-light` | up to 2 CPU / 4Gi | no | lint, typecheck, unit tests, git tags, ArgoCD restarts, registry cleanup |
| `self-hosted-heavy` | up to 4 CPU / 8Gi | DinD sidecar (`tcp://localhost:2375`) | Docker builds, E2E, Go race tests, multi-image builds |

Runners are **ephemeral** (fresh pod per job) and scale from zero — expect a
~30–60s cold start when no runner is warm. `minRunners: 0` keeps the cluster
cost at zero when idle.

## How routing works — scale-set scoping

ARC scale sets are registered against a `githubConfigUrl`. A job can only be
picked up if a scale set with a matching name is registered **for that repo**
(repo-scoped) or **for the org** (org-scoped).

Current state (deployed via `infra.env/applications/test/shared/github-runners/`):

| Scale set | Scope | Label |
|---|---|---|
| `game-engine-heavy` | repo `gatewaynet/game-engine` | `self-hosted-heavy` |
| `game-engine-regulated-heavy` | repo `gatewaynet/game-engine-regulated` | `self-hosted-heavy` |
| `platform-engine-light` / `-heavy` | repo `gatewaynet/platform-engine` | `self-hosted-light` / `-heavy` |
| `affiliate-backend` | repo `gatewaynet/affiliate-backend` | `self-hosted-heavy` |

**A repo without a covering scale set that targets `runs-on:
self-hosted-*` will queue forever.**

> **⚠️ Important — `gatewaynet` is a personal user account, not an
> organization.** GitHub has no user-level runner registration, so
> **org-wide scale sets are not possible today**: every repo needs its own
> repo-scoped scale set pair. The org-wide values in
> `runners/runner-scale-sets/` only become deployable if the account is
> converted to an organization (GitHub Settings → Organizations → "Turn
> gatewaynet into an organization") — which would also unlock org variables,
> runner groups, and org secrets.

## Getting runner coverage for your repo

Runner deployments are GitOps-managed in `infra.env` — do **not** helm-install
by hand. Open an issue on `gatewaynet/infra.env` (title format
`[add][github-runners] ...`) asking for coverage, referencing the values in
`runners/runner-scale-sets/` here. Deployment lands under
`infra.env/applications/test/shared/github-runners/`.

The GitHub App credentials live in the `github-arc-credentials` secret in the
`github-runners` namespace. Org-scoped scale sets require the App to have
org-level **Self-hosted runners: read & write**.

## Writing jobs for these runners

The runner image is `ghcr.io/actions/actions-runner` — much slimmer than
GitHub's `ubuntu-latest`:

- **No node/npm on PATH** — always use `actions/setup-node` before running
  npm/node commands.
- **`/usr/local/bin` is root-owned** — install tools into `$RUNNER_TEMP` and
  append to `$GITHUB_PATH` (see `actions/argocd-restart` for the pattern), or
  use `sudo` (passwordless sudo is available).
- **Docker only on heavy** — `docker`/`buildx` CLIs are present on the image,
  but a daemon only exists on `self-hosted-heavy` (DinD sidecar). Any
  `docker build`/`docker run` on light will fail.
- git, curl, jq, tar are available; GNU grep with `-P` works.
- Egress: runners reach the DO registry, `argocd.clanroot.dev` and
  `argocd.clanroot.net` normally.

## Outage failover

If k8s-xeon is down, the failover watchdog sets `CI_RUNNER_OVERRIDE` on every repo to move every
stage to GitHub-hosted runners (or
`ubuntu-latest`) with a single switch. Full runbook:
[002_CI_FAILOVER.md](002_CI_FAILOVER.md).

## Operations cheat-sheet

```bash
# Everything below runs against the test cluster
kubectl --context k8s-xeon -n github-runners get pods          # controller, listeners, active runners
kubectl --context k8s-xeon -n github-runners get autoscalingrunnersets
kubectl --context k8s-xeon -n github-runners logs deploy/arc-controller-gha-rs-controller
helm --kube-context k8s-xeon -n github-runners list
```

Troubleshooting:

| Symptom | Likely cause |
|---|---|
| Job queued forever ("Waiting for a runner") | No scale set covering this repo with that label — see scoping above |
| `docker: command failed` / cannot connect to daemon | Job ran on light runner; move to `self-hosted-heavy` |
| `node: command not found` | Missing `actions/setup-node` step |
| `Permission denied` writing `/usr/local/bin` | Use `$RUNNER_TEMP` + `$GITHUB_PATH`, or `sudo` |
| All jobs of one repo pending, listener pod missing | Scale set not deployed / GitHub App auth broken — check controller logs |
