# Backup Runner Droplet — CI Failover Plan

Goal: if the k8s-xeon cluster (which hosts all ARC runners) goes offline, CI
builds keep working. Design: **active-passive failover**, not dual
registration.

## Why not register a droplet with the same labels?

ARC scale sets route jobs by scale-set *name* (`self-hosted-light/heavy`).
GitHub explicitly warns against classic runners sharing names/labels with
scale sets — routing becomes unpredictable and jobs can starve. So the backup
runner uses its **own label** (`self-hosted-backup`) and stays idle until we
point traffic at it.

## The failover switch — `CI_RUNNER_OVERRIDE`

Every `runs-on` in the shared workflows resolves as:

```yaml
runs-on: ${{ vars.CI_RUNNER_OVERRIDE || inputs.<stage>-runner }}
```

`vars` resolves from the calling repo, falling back to the **org-level**
variable. Failover for the whole org is therefore:

```bash
# k8s-xeon down → move ALL stages of ALL repos to the backup droplet
gh api -X POST orgs/gatewaynet/actions/variables \
  -f name=CI_RUNNER_OVERRIDE -f value=self-hosted-backup -f visibility=all
# (use -X PATCH orgs/gatewaynet/actions/variables/CI_RUNNER_OVERRIDE if it exists)

# recovery → delete the variable, traffic returns to the k8s runners
gh api -X DELETE orgs/gatewaynet/actions/variables/CI_RUNNER_OVERRIDE
```

Emergency alternative with zero self-hosted capacity: set the value to
`ubuntu-latest` (billed GitHub-hosted minutes, but zero setup).

## Droplet specification (proposed)

| Item | Value |
|---|---|
| Size | `s-4vcpu-8gb` (~$48/mo) — matches the heavy runner class |
| Image | Ubuntu 24.04 LTS |
| Region | `ams3` (same as clusters/registry) |
| Software | Docker CE + buildx, GitHub Actions runner (classic, org-level) |
| Labels | `self-hosted-backup` |
| Runner scope | Org (`gatewaynet`) → default runner group (all repos) |

One droplet can register 2 runner instances (one light-ish, one build) if
parallelism during an outage matters; start with one.

## Provisioning (when approved)

```bash
doctl compute droplet create ci-backup-runner \
  --size s-4vcpu-8gb --image ubuntu-24-04-x64 --region ams3 \
  --ssh-keys <key-id> --tag-names ci,runner

# On the droplet:
apt-get update && apt-get install -y docker.io docker-buildx git jq curl
useradd -m runner && usermod -aG docker runner

# Register org-level classic runner (token from:
#   gh api -X POST orgs/gatewaynet/actions/runners/registration-token -q .token)
sudo -iu runner bash -c '
  mkdir actions-runner && cd actions-runner
  curl -o runner.tar.gz -L https://github.com/actions/runner/releases/latest/download/actions-runner-linux-x64-<ver>.tar.gz
  tar xzf runner.tar.gz
  ./config.sh --url https://github.com/gatewaynet --token <REG_TOKEN> \
    --name ci-backup-runner --labels self-hosted-backup --unattended
'
cd /home/runner/actions-runner && ./svc.sh install runner && ./svc.sh start
```

Notes:
- A classic runner is **not ephemeral** — workspace persists between jobs.
  Acceptable for a break-glass backup; do not use it as primary.
- Docker daemon is local, so `docker-build-push` works as-is.
- The runner is org-scoped: available to every repo without per-repo setup.

## Cost options

| Option | Cost | Failover time |
|---|---|---|
| Droplet always on (warm standby) | ~$48/mo | seconds (flip the variable) |
| Droplet powered off | still billed by DO | ~2 min (boot + flip) |
| Snapshot only, create on demand | ~$0.3/mo storage | ~10 min (create from snapshot + new reg token + flip) |
| `CI_RUNNER_OVERRIDE=ubuntu-latest` | GH minutes during outage only | seconds |

Recommendation: **snapshot-on-demand + `ubuntu-latest` as the immediate
stopgap** — flip to `ubuntu-latest` the moment k8s-xeon dies (builds keep
flowing, paying GH minutes briefly), restore the droplet from snapshot, then
flip to `self-hosted-backup`. Zero standing cost, minutes of billed CI at
most.

## Status

- [x] Failover switch (`CI_RUNNER_OVERRIDE`) wired into `node-ci-cd.yml` and `pr-cleanup.yml`
- [ ] Droplet/snapshot provisioned (awaiting approval — pick a cost option above)
- [ ] Failover drill: flip variable, run a build, flip back
