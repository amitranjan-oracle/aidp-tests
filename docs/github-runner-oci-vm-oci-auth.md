# Letting runner jobs call OCI without stored credentials

Companion to [github-runner-oci-vm.md](./github-runner-oci-vm.md).

Because the runner is an OCI compute instance, workflows can authenticate to OCI
services as the **instance itself** — an *instance principal*. No API keys in
GitHub secrets, no key files on disk, nothing to rotate. This is the main
practical reason to run CI on an OCI VM rather than a hosted runner.

The OCI CLI and, if you use it, the OCI Python SDK must already be installed on
the VM — neither ships with the runner.

## 1. Create a dynamic group matching the VM

```
ALL {instance.id = 'ocid1.instance.oc1..<your-instance>'}
```

A compartment-scoped rule is easier to maintain but broader — every instance in
that compartment gains the same rights, including ones added later:

```
ALL {instance.compartment.id = 'ocid1.compartment.oc1..<compartment>'}
```

## 2. Grant it only what CI needs

```
Allow dynamic-group <DG-NAME> to use <service>-family in compartment <COMPARTMENT>
```

Every job on this runner inherits these permissions, so scope them to the
compartments and services your pipeline actually touches, and prefer several
narrow statements over one broad one. Pick the weakest verb that does the job —
`inspect` < `read` < `use` < `manage`. `manage` is full administrative control
over every resource in the family, which a deploy pipeline rarely needs.

## 3. Verify from the VM

```bash
OCI_CLI_AUTH=instance_principal \
  oci iam availability-domain list --compartment-id <COMPARTMENT_OCID>
```

In Python, the signer constructor confirms the instance can obtain an identity
at all:

```python
import oci
oci.auth.signers.InstancePrincipalsSecurityTokenSigner()
```

Be clear on what each check proves. The constructor only federates the
instance's certificate into a security token — it succeeds even with no dynamic
group and no policy, because membership and permissions are evaluated when a
*request* is authorized, not when the signer is built. So a constructor failure
points at the instance metadata service or the certificate, whereas a
`NotAuthorizedOrNotFound` from the CLI call above is what a missing
dynamic-group rule or policy actually looks like.

## 4. Use it in a workflow

```yaml
      - name: Check OCI access
        env:
          OCI_CLI_AUTH: instance_principal
        run: oci iam availability-domain list --compartment-id <COMPARTMENT_OCID>
```

This proves the instance principal works from inside a job, and nothing more —
it does not prove your pipeline's real calls are permitted. Once it passes,
replace it with a command against the service you actually use, in the same
compartment, so the check exercises the policy from step 2.

Either way: no `oci setup config`, no private key, no GitHub secret.

## Caveat: identity context

Some services resolve resources in the **caller's** identity context, so an
object created interactively by a human user can be invisible to the instance
principal, and vice versa.

We hit this with Oracle AI Data Platform git credentials: *in our own testing*, a
credential created by a user account was not visible to the instance principal,
and git operations failed server-side with a generic error rather than a
permissions message. Treat that as an observation from one environment rather
than documented product behaviour — but it's a useful debugging lead.

The general point holds regardless: if a CI call fails where the identical call
succeeds from your laptop, check **which identity created the resource** before
assuming a permissions bug.

## Reference

- [Calling OCI services from an instance](https://docs.oracle.com/en-us/iaas/Content/Identity/Tasks/callingservicesfrominstances.htm)
- [Managing dynamic groups](https://docs.oracle.com/en-us/iaas/Content/Identity/dynamicgroups/managingdynamicgroups.htm)
