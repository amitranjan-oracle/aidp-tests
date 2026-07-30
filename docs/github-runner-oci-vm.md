# GitHub Actions self-hosted runner on an OCI Linux VM

How to run your GitHub Actions workflows on your own Oracle Cloud
Infrastructure compute instance, inside your own network.

Written for **Oracle Linux 9 (x86_64)** and runner **v2.336.0**. Replace
`<ORG>`, `<REPO>`, `<runner-name>` and OCIDs with your own values.

Two companion pages, each needed only in its own case:
[private GitHub Enterprise Server](./github-runner-oci-vm-ghes.md) ·
[calling OCI without stored credentials](./github-runner-oci-vm-oci-auth.md).

## How it works

The runner is a small agent on your VM. It opens an **outbound** HTTPS long-poll
to GitHub, and queued jobs are pushed down that existing connection.

- No inbound port, no public IP, no SSH from GitHub into your VM.
- The VM can sit in a **private subnet**; it needs only outbound 443.
- Jobs run as a local OS user, so they can reach private resources and use the
  VM's own OCI identity.

## Prerequisites

| Item | Requirement |
| --- | --- |
| VM | Oracle Linux 9, x86_64, already provisioned. 2 OCPU / 16 GB and a 100 GB boot volume is a comfortable start; size to your jobs. |
| OS access | A user with `sudo` (`opc` on OCI images). Do **not** run the runner as `root`. |
| Egress | Outbound TCP 443 to GitHub. In a private subnet, route `0.0.0.0/0` via a **NAT gateway** — a service gateway is not enough, it reaches only OCI services. |
| Ingress | None. |
| GitHub | **Admin** on the target repository. |
| Job runtime | Tools your workflows invoke (`git`, `python3`, `docker`, OCI CLI, …) are **not** bundled with the runner. Install them yourself. |

Confirm egress from the VM first:

```bash
curl -sS -o /dev/null -w '%{http_code}\n' https://api.github.com   # expect 200
```

## Step 1 — Get a registration token

**Single-use, expires in about an hour** — fetch it when you're ready for
Step 3. Console: repo → **Settings → Actions → Runners → New self-hosted
runner**. Or, as a repo admin:

```bash
gh api --method POST \
  repos/<ORG>/<REPO>/actions/runners/registration-token --jq .token
```

Treat it as a credential: don't commit it or paste it into a ticket.

## Step 2 — Install under `/opt`

Install into **`/opt/actions-runner`, not a home directory.** On Oracle Linux 9
with SELinux enforcing, systemd cannot exec the runner's launcher scripts from
under `/home`: the service installs, then fails to start with
`203/EXEC … runsvc.sh: Permission denied`. `/opt` is labelled to allow it.

Run these as `opc`, using `sudo` only where shown — never run the runner itself
as `root`. Every command below assumes `/opt/actions-runner` is your working
directory, as set on the second line:

```bash
sudo mkdir -p /opt/actions-runner && sudo chown opc:opc /opt/actions-runner
cd /opt/actions-runner

RUNNER_VERSION=2.336.0          # current at time of writing
curl -fsSL -o runner.tar.gz \
  "https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz"

# verify against the SHA-256 on that release's page
echo "<SHA256_FROM_RELEASE_PAGE>  runner.tar.gz" | sha256sum -c -

tar xzf runner.tar.gz && rm runner.tar.gz
sudo ./bin/installdependencies.sh    # OL9: libicu, openssl-libs, krb5-libs, zlib, lttng-ust
```

For the current version and checksum:
`gh api repos/actions/runner/releases/latest --jq .tag_name`, or the
[releases page](https://github.com/actions/runner/releases).

## Step 3 — Register with your repository

```bash
cd /opt/actions-runner
./config.sh \
  --url https://github.com/<ORG>/<REPO> \
  --token <REGISTRATION_TOKEN> \
  --name <runner-name> \
  --labels self-hosted,oci,vm \
  --unattended --replace
```

Expect `√ Runner successfully added`.

**Labels are how workflows select this runner** — those labels mean a job targets
it with `runs-on: [self-hosted, oci, vm]`. A job whose `runs-on` labels are not
*all* present on some online runner stays **Queued** indefinitely, so give each
runner a distinguishing label (e.g. `vm` vs `oke`) to steer jobs later.

`--replace` lets you re-register the same `--name` without deleting the old
entry first — handy when rebuilding the VM.

## Step 4 — Run it as a systemd service

```bash
cd /opt/actions-runner
sudo ./svc.sh install opc     # OS user the runner, and every job, runs as
sudo ./svc.sh start
sudo ./svc.sh status          # expect: active (running)
```

Enabled at boot, so the runner returns after a reboot. Logs — either should show
`Listening for Jobs`:

```bash
sudo journalctl -u 'actions.runner.*' -f          # service level
tail -f /opt/actions-runner/_diag/Runner_*.log    # runner detail
```

> Already installed under `/home` and `start` fails `203/EXEC`? That code is
> systemd's generic "could not exec", so confirm the cause before acting on it:
> `sudo ausearch -m avc -ts recent` shows an SELinux denial if that's what this
> is, and `ls -Z /home/opc/actions-runner/runsvc.sh` shows the label
> (`user_home_t` is one systemd won't exec). With no denial recorded, look
> instead at the execute bit, a `noexec` mount, or the shebang. For a genuine
> label problem, either move the install to `/opt`, or relabel in place:
> `sudo semanage fcontext -a -t bin_t '/home/opc/actions-runner(/.*)?'` then
> `sudo restorecon -R /home/opc/actions-runner`. Note the `(/.*)?` — a bare
> `/.*` matches only the contents, leaving the directory itself mislabelled.

Long-lived runners self-update to newer runner releases, so you won't normally
repeat Step 2.

## Step 5 — Verify end to end

The runner should show **Idle** under repo → **Settings → Actions → Runners**.
Commit this as `.github/workflows/runner-smoke.yml` **on your default branch**,
then run **Actions → runner-smoke → Run workflow**:

```yaml
name: runner-smoke
on: workflow_dispatch
jobs:
  smoke:
    runs-on: [self-hosted, oci, vm]     # must match your --labels
    steps:
      - run: hostname; uname -r; whoami; pwd
```

Two common snags:

- `workflow_dispatch` appears only once the file exists on the **default
  branch**. From a feature branch you get
  `404: workflow … not found on the default branch`.
- Pushing anything under `.github/workflows/` over HTTPS needs a token with the
  **`workflow`** scope, or the push is refused. Either
  `gh auth refresh -h github.com -s workflow`, or push over SSH — SSH keys are
  exempt.

## Security

A self-hosted runner executes workflow code on your machine, with your network
position and your cloud identity. Treat it as production infrastructure.

- **Don't attach one to a public repository** unless you control exactly which
  workflows run — a `pull_request` trigger from a fork would execute a
  stranger's code on your VM. If the repo must be public, restrict triggers to
  `push` / `workflow_dispatch` on trusted branches and require approval for
  outside contributors. This is
  [GitHub's own guidance](https://docs.github.com/en/actions/reference/security/secure-use).
- **Dedicate the VM.** A job reads anything the runner user can read, including
  earlier jobs' leftover files. Don't co-host unrelated workloads.
- **Stay non-root** — install and run as `opc`, never `sudo ./run.sh`.
- **Least privilege on OCI.** Every job inherits whatever IAM the VM has; scope
  it to what CI actually touches.
- **Keep the network tight** — private subnet, NAT for egress, no ingress rules,
  NSGs so only the runner reaches your databases.

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Runner shows **Offline** | service stopped, or VM down | `cd /opt/actions-runner && sudo ./svc.sh status`; `sudo journalctl -u 'actions.runner.*' -n 50` |
| `203/EXEC … Permission denied` at start | SELinux blocking systemd exec under `/home` | install under `/opt` (Step 2) |
| `config.sh` rejects the token | token expired (~1 h) or already used | mint a fresh one (Step 1) |
| `config.sh` hangs, or TLS errors | egress blocked | `curl https://api.github.com`; check NAT gateway, route table, security list / NSG, proxy |
| `installdependencies.sh` fails | no dnf mirror reachable from a private subnet | allow egress to the mirror, or use a local repo |
| Job stays **Queued** | `runs-on` labels match no online runner | compare `runs-on` with the runner's labels in the UI |
| Job fails `command not found` | tool not on the VM | install it; the runner ships no toolchain |

Behind an HTTP proxy, the runner reads `https_proxy` / `no_proxy` from a `.env`
file beside `config.sh` — see
[Using proxy servers](https://docs.github.com/en/actions/how-tos/manage-runners/use-proxy-servers).

## Further reading

- [About self-hosted runners](https://docs.github.com/en/actions/concepts/runners/self-hosted-runners)
  · [Adding self-hosted runners](https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/add-runners)
- [Configuring the runner as a service](https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/configure-the-application)
  · [Monitoring and troubleshooting](https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/monitor-and-troubleshoot)
- [Security hardening for GitHub Actions](https://docs.github.com/en/actions/reference/security/secure-use)
- [NAT gateway](https://docs.oracle.com/en-us/iaas/Content/Network/Tasks/NATgateway.htm)
  · [Calling OCI services from an instance](https://docs.oracle.com/en-us/iaas/Content/Identity/Tasks/callingservicesfrominstances.htm)
