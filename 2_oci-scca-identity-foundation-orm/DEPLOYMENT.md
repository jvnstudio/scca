# Deploy Stack 02: OCI SCCA Identity Foundation

This stack creates one OCI identity domain in the Stack 01 VDMS compartment and publishes a human-group catalog for later access-policy work. It does **not** create users, memberships, policies, federation, credentials, MFA/sign-on policies, or AD Bridge resources.

## 1. Prerequisites and ownership

Before deployment, obtain:

- The `vdms_compartment_ocid` output from Stack 01.
- The approved OCI identity-domain home region, for example `us-ashburn-1`.
- The corresponding naming key, for example `IAD`, and deployment label, for example `PROD`.
- An OCI Resource Manager stack-storage compartment.
- IAM permission for the Resource Manager job to create and manage identity domains in the VDMS compartment.

For production, confirm these responsibilities before creating the stack:

- On-premises AD owns human users and groups.
- AD Bridge performs approved **inbound** synchronization from AD to OCI.
- AD FS handles federated authentication.
- AD administrators' YubiKey and PIV/CAC requirements are enforced in AD FS/on-premises authentication policy and, where approved, a later OCI identity-provider/sign-on-policy change. This stack cannot enforce those authentication factors.

Keep the two approved cloud-local break-glass accounts in OCI's Default identity domain. Do not synchronize or manage them with this stack.

## 2. Verify the configuration locally

Use Terraform 1.5.7, the version supported by OCI Resource Manager.

```bash
cd /Users/johnvngt/Github/scca/2_oci-scca-identity-foundation-orm
tfenv install 1.5.7
tfenv use 1.5.7
terraform init -backend=false
terraform fmt -check -recursive
terraform validate
```

Keep the generated `.terraform.lock.hcl`; it records the tested OCI provider version. Do not commit `.terraform/` because it contains downloaded provider binaries.

## 3. Set deployment values

Copy the example for local reference:

```bash
cp terraform.tfvars.example terraform.tfvars
```

Set the required values:

```hcl
vdms_compartment_ocid = "ocid1.compartment..." # Stack 01 VDMS output
region                = "us-ashburn-1"
region_key            = "IAD"
resource_label        = "PROD"

identity_domain_license_type = "free"
group_provisioning_mode      = "AD_SYNC"
workload_identifiers         = ["SWR1", "SWR2"]
additional_group_names       = []
```

Production requirements:

- Keep `group_provisioning_mode = "AD_SYNC"`.
- Keep `identity_domain_license_type = "free"` unless Premium is formally approved.
- Give the AD team `ad-group-manifest.csv`; the default catalog contains 20 groups, including ENTOPS, CLOUDOPS, SECOPS, APPDEV, and DEVOPS.
- If you add a workload identifier or extra group, add the matching on-prem AD group to the approved provisioning process before synchronization.

`OCI_NATIVE` is for isolated labs only. Never use it in a domain that will synchronize same-named groups from AD.

## 4. Build the Resource Manager ZIP

Create an archive with Terraform files at the archive root:

```bash
cd /Users/johnvngt/Github/scca/2_oci-scca-identity-foundation-orm
zip -r ../oci-scca-identity-foundation-orm.zip . \
  -x '.terraform/*' \
  -x 'terraform.tfvars' \
  -x '*.tfplan' \
  -x '.DS_Store'
unzip -l ../oci-scca-identity-foundation-orm.zip
```

Do not package real variable files, provider binaries, secrets, API keys, private keys, AD credentials, or the AD Bridge agent configuration.

## 5. Create the Resource Manager stack

1. In OCI Console, go to **Developer Services → Resource Manager → Stacks**.
2. Select **Create stack**, then **My configuration**.
3. Upload `oci-scca-identity-foundation-orm.zip`.
4. Select the compartment that stores the Resource Manager stack. This is separate from the VDMS compartment where the identity domain is created.
5. Select Terraform version **1.5.x**.
6. Enter the values from section 3.
7. Create the stack without selecting **Run apply**.

## 6. Plan and apply safely

1. Create a **Plan** job.
2. In production `AD_SYNC` mode, confirm the plan contains exactly one resource to add:

   ```text
   oci_identity_domain.scca
   ```

3. Stop if the plan contains any `oci_identity_domains_group.human` resource. That indicates `OCI_NATIVE` mode and is not permitted for the AD-synchronized production domain.
4. Confirm the VDMS compartment OCID, identity-domain display name, home region, and Free license selection.
5. After approval, apply the reviewed plan.

## 7. Configure AD synchronization and federation after apply

After the identity-domain apply succeeds:

1. Give `ad-group-manifest.csv` to the AD administrators.
2. Create/verify each approved group in the designated on-prem AD OU. Manage membership through AD governance.
3. Configure AD Bridge to synchronize approved users and groups **from AD to OCI**.
4. Keep AD authoritative: do not enable Bridge actions that activate/deactivate users in AD, update user attributes in AD, or update groups in AD.
5. Configure AD FS federation in its approved, separate change window.
6. Verify that synchronized group names appear in the new OCI identity domain.
7. Verify AD-admin sign-in requires the approved YubiKey and PIV/CAC controls in AD FS/on-premises policy before enabling privileged access mappings.
8. Only after synchronization and federation are verified, deploy a later least-privilege IAM policy stack to map groups to approved permissions.

## 8. Ongoing changes and retirement

- Add or change groups in AD first, update this stack's catalog inputs/manifest, run Plan, then Apply only after review.
- Do not switch an established production domain from `AD_SYNC` to `OCI_NATIVE`.
- Do not destroy the identity domain as a routine rollback. It can affect synchronized identities and applications. Follow the approved identity-domain decommissioning procedure instead.
