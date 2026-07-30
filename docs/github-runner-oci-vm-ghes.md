# Registering the runner against GitHub Enterprise Server

Companion to [github-runner-oci-vm.md](./github-runner-oci-vm.md). Read this
only if your GitHub is a private **GitHub Enterprise Server (GHES)** appliance
— in your own datacenter, or in a VCN — rather than github.com.

The connection model is unchanged: the runner still dials **outbound** and needs
no inbound port. What changes is that the target hostname is private, so three
things must be true before `config.sh` can succeed.

**Substitute these in the main guide before you start.** Three of its steps
assume github.com and will fail against an isolated appliance:

| Main guide | Replace with |
| --- | --- |
| Egress check `curl https://api.github.com` | The same check against your GHES host |
| Runner tarball from github.com | The download URL your appliance's *New self-hosted runner* page shows — often the only source reachable |
| Registration token from github.com | The token from GHES (see [Register](#register) below) |

## 1. Network path

The VM must be able to route to the GHES address:

| GHES location | What you need |
| --- | --- |
| Another VCN, same region | Local VCN peering, plus route rules and security rules on both sides |
| Another VCN, another region | Remote VCN peering via DRGs |
| On-premises | Site-to-Site VPN or FastConnect |
| Same VCN | Nothing routing-wise — intra-VCN traffic routes implicitly; you need only the security rules below |

In every case the runner's subnet needs **egress** on 443, and the GHES subnet
needs matching **ingress** on 443.

Test by **IP address**, not hostname. DNS is step 2, and testing by name here
would report a name-resolution failure as a network-path failure:

```bash
nc -vz <ghes-ip> 443
```

## 2. DNS resolution

The VM must resolve the GHES FQDN; public DNS won't have it. Options, best
first:

- An OCI **private DNS zone** attached to the VCN's resolver, with an A record
  for the FQDN.
- A **forwarding rule** on the VCN resolver pointing at your corporate DNS.
- An `/etc/hosts` entry — last resort, single host, easy to forget later.

```bash
getent hosts <ghes-host>
```

Use the FQDN the certificate was issued for in `--url`. An IP address or short
name resolves but then fails TLS verification.

## 3. TLS trust

If GHES presents a certificate from an internal CA, add that CA to the OS trust
store. Get the certificate from whoever runs your PKI. It must be **PEM**
(begins `-----BEGIN CERTIFICATE-----`, not DER) and include any intermediates in
the chain, or verification still fails:

```bash
sudo cp internal-ca.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust extract
curl -sSI https://<ghes-host> >/dev/null && echo "TLS OK"
```

Node-based actions (anything `uses:`-ing a JavaScript action) carry their own CA
bundle and will still fail after the step above. Point them at the CA too, via a
`.env` file beside `config.sh`:

```bash
echo 'NODE_EXTRA_CA_CERTS=/etc/pki/ca-trust/source/anchors/internal-ca.crt' \
  >> /opt/actions-runner/.env
cd /opt/actions-runner                        # svc.sh only works from here
sudo ./svc.sh stop && sudo ./svc.sh start     # .env is read at start
```

## Register

Identical to Step 3 of the main guide, with the GHES URL:

```bash
cd /opt/actions-runner
./config.sh \
  --url https://<ghes-host>/<ORG>/<REPO> \
  --token <REGISTRATION_TOKEN> \
  --name <runner-name> \
  --labels self-hosted,oci,vm \
  --unattended --replace
```

Get the token from GHES itself: repo → **Settings → Actions → Runners → New
self-hosted runner**. That page also shows the exact download URL and version
your appliance expects.

## Runner version compatibility

Each GHES release supports a bounded range of runner versions, and runners
connected to GHES pull their auto-updates **from the appliance**, not from
github.com. Prefer the version offered on your appliance's *New self-hosted
runner* page over the newest release in `actions/runner`.

## Troubleshooting

| Symptom | Cause |
| --- | --- |
| `config.sh` → `Could not resolve host` | DNS (§2) |
| `nc` to the IP times out, no TLS handshake | no network path (§1) |
| `SSL certificate problem: unable to get local issuer certificate` | internal CA not trusted, or PEM chain incomplete (§3) |
| `curl` works but JavaScript actions fail TLS | `NODE_EXTRA_CA_CERTS` not set (§3) |
| Runner registers, then fails on auto-update | runner/GHES version mismatch (see above) |

Reference: your appliance's own docs at
`https://<ghes-host>/en/actions`, or the versioned public copy —
[GHES: adding self-hosted runners](https://docs.github.com/en/enterprise-server@3.17/actions/how-tos/manage-runners/self-hosted-runners/add-runners)
(swap the version to match your appliance).
