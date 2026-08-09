# CI Failover — k8s-xeon Offline Runbook

All runners live on the k8s-xeon cluster, which sits behind a single uplink.
When that connection drops, no self-hosted runner can pick up jobs. The
failover is **GitHub-hosted runners, toggled by one org variable** — zero
standing cost, billed minutes only while the outage lasts. Since all normal
CI is self-hosted, the Team plan's 3000 included minutes/month are entirely
available for the watchdog and outage-mode builds.

## Automatic failover — the watchdog

`.github/workflows/runner-failover.yml` (this repo) probes
`https://argocd.clanroot.dev` from a GitHub-hosted runner every 30 minutes
(two attempts, 30s apart — any HTTP response counts as up, only connection
failure/timeout as down):

- **Cluster offline** → sets the repo variable
  `CI_RUNNER_OVERRIDE=ubuntu-latest` on **every owned repo** — all pipelines
  fail over. (`gatewaynet` is a personal user account, so there are no org
  variables; the watchdog reconciles per-repo instead.)
- **While offline** → sweeps every repo for queued runs whose jobs wait
  on `self-hosted-*` labels (queued before the flip, would never start) and
  cancel+reruns them so they re-resolve onto `ubuntu-latest`. Repeats each
  tick to catch stragglers.
- **Cluster back** → deletes the variable from every repo where its value is
  exactly `ubuntu-latest` — traffic returns to self-hosted. A manually-set
  override with any other value (e.g. `ubuntu-24.04` — same GitHub-hosted
  image, different label) is never touched: use that to force failover
  manually without the watchdog reverting it.

Detection latency is ≤ ~30 min; cost ≈ 1440 included minutes/month. Tighten
the cron to `*/15` for ≤ ~15 min latency (≈ 2880 min/month — nearly the whole
free allotment).

**One-time setup (done 2026-08-09):** a fine-grained PAT (owner `gatewaynet`,
access to **All repositories**, Repository permissions **Actions: read &
write** + **Variables: read & write**) stored as the `CI_FAILOVER_TOKEN`
secret in this repo. Expires **2026-09-08** — watchdog runs turn red when it
lapses; rotate then. Without the secret the watchdog runs neutrally and does
nothing.

**Limitation:** the sweep targets runs whose overall status is `queued`. A
run caught mid-pipeline (early jobs finished before the outage, later jobs
queued) shows as `in_progress` and is left alone — re-run it manually if it
matters during the outage; otherwise it resumes when the cluster returns.

## Manual switch — `CI_RUNNER_OVERRIDE`

Every `runs-on` in the shared workflows resolves as:

```yaml
runs-on: ${{ vars.CI_RUNNER_OVERRIDE || inputs.<stage>-runner }}
```

`vars` resolves from the calling repo's variables (`gatewaynet` is a user
account — there is no org-level fallback).

The watchdog handles the switch automatically. To force it by hand, prefer
triggering the watchdog (`gh workflow run runner-failover.yml -R
gatewaynet/ci-workflows`) — it reconciles based on the actual probe. To force
failover **while the cluster is up** (e.g. runners misbehaving but reachable),
use a value the watchdog won't auto-revert:

```bash
# force (per affected repo; ubuntu-24.04 = GitHub-hosted, watchdog leaves it alone)
gh variable set CI_RUNNER_OVERRIDE -b ubuntu-24.04 -R gatewaynet/<repo>
# undo
gh variable delete CI_RUNNER_OVERRIDE -R gatewaynet/<repo>
```

## What works during failover

- Checks and Docker builds: fully — `ubuntu-latest` ships node and docker.
- Registry pushes: fully — DO registry is unaffected by the cluster being down.
- Git tags / version sync / PR comments: fully.
- **ArgoCD test restarts: fail** (test ArgoCD runs on the offline cluster).
  These jobs are `continue-on-error`, so pipelines stay green; images wait in
  the registry and deploy when the cluster returns (re-run the restart job or
  let the next build restart it).

## Cost

GitHub-hosted minutes for private repos bill at ~$0.008/min (Linux). A typical
outage-day of builds is a few dollars — far below any always-on backup
runner. This is why the droplet option below was rejected.

## Rejected alternative — backup droplet

Considered 2026-08: a DO droplet (`s-4vcpu-8gb`, ~$48/mo each, 2× for HA)
registered as a classic runner with label `self-hosted-backup`.
Rejected: standing cost exceeds the realistic spend on GitHub-hosted minutes
during rare connectivity outages. If outage frequency ever changes, the
sketch: Ubuntu + Docker CE + classic runner as systemd service, org-scoped,
own label (never reuse the ARC scale-set labels — GitHub warns duplicate
names across scopes cause unpredictable routing), then point
`CI_RUNNER_OVERRIDE=self-hosted-backup` at it.
