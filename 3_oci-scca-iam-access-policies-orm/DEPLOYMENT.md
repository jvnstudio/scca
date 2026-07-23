# Deploy Stack 03: OCI SCCA IAM Access Policies

This OCI Resource Manager stack creates only `oci_identity_policy` resources. It maps the AD-synchronized human groups cataloged in Stack 02 to narrowly scoped OCI permissions for the compartments created by Stack 01.

It does not create or modify users, groups, memberships, dynamic groups, identity providers, federation, credentials, MFA rules, or AD Bridge configuration.

## 1. Prerequisites and approval boundary

Deploy Stacks 01 and 02 first. Collect these outputs:

- Stack 01: `home_compartment_ocid`, `vdms_compartment_ocid`, `vdss_compartment_ocid`, and `workload_compartment_ocids_json`.
- Stack 02: `identity_domain_display_name`.

You also need:

- The tenancy OCID.
- The OCI region, region key, and deployment label.
- The compartment that stores the Resource Manager stacks. If it differs from the SCCA Home compartment, record its OCID for `orm_compartment_ocid`.
- Approved values for each functional-role workload assignment.
- A reviewed IAM policy that authorizes the Resource Manager job identity to manage policies in the tenancy and SCCA Home-compartment scopes.

The final prerequisite is mandatory. This stack creates policy objects, including three optional tenancy-scoped policies. Resource Manager cannot grant itself the authority to create them.

## 2. Verify locally

Use Terraform 1.5.7, the OCI Resource Manager-supported version.

```bash
cd /Users/johnvngt/Github/scca/3_oci-scca-iam-access-policies-orm
tfenv install 1.5.7
tfenv use 1.5.7
terraform init -backend=false
terraform fmt -check -recursive
terraform validate
```

Retain `.terraform.lock.hcl` to record the tested provider version. Do not commit `.terraform/`, real variable files, or plan files.

## 3. Set the inputs

Copy the supplied example for local reference:

```bash
cp terraform.tfvars.example terraform.tfvars
```

Set the Stack 01 and Stack 02 outputs exactly as issued:

```hcl
tenancy_ocid                   = "ocid1.tenancy..."
home_compartment_ocid          = "ocid1.compartment..."
vdms_compartment_ocid          = "ocid1.compartment..."
vdss_compartment_ocid          = "ocid1.compartment..."
orm_compartment_ocid           = "" # Empty means the SCCA Home compartment.
workload_compartment_ocids_json = "{\"SWR1\":\"ocid1.compartment...\",\"SWR2\":\"ocid1.compartment...\"}"

region                       = "us-ashburn-1"
region_key                   = "IAD"
resource_label               = "PROD"
identity_domain_display_name = "OCI-SCCA-LZ-Domain-IAD-PROD"
```

Functional roles receive no workload access by default. Grant access only after approval, using an existing workload key:

```hcl
cloudops_workload_keys        = ["SWR1"]
appdev_workload_keys          = []
devops_workload_keys          = ["SWR1"]
database_admin_workload_keys  = []
storage_admin_workload_keys   = ["SWR2"]
backup_admin_workload_keys    = []
```

Every assignment key must occur in `workload_compartment_ocids_json`. A nonexistent key causes the plan to fail. Keep `enable_tenancy_scope_policies = true` unless the required root statements are deployed by an approved separate process.

## 4. Review the intended permission model

Before packaging, review `policy-matrix.csv` and the rendered groups in `locals.tf`.

The stack intentionally does not grant:

- `manage all-resources`;
- IAM policy, user, group, dynamic-group, domain, key, or secret management;
- Resource Manager Destroy jobs;
- implicit functional workload access; or
- OCI permissions to Helpdesk-Operators.

Workload administrators are confined to their matching workload compartment. CLOUDOPS, APPDEV, DEVOPS, Database-Admins, Storage-Admins, and Backup-Admins receive permissions only in the explicitly listed workload compartments.

AD-admin authentication factors remain controlled by AD FS/on-premises policy and later identity-domain sign-on configuration; this IAM stack does not configure YubiKey, PIV/CAC, federation, or MFA.

## 5. Build the upload ZIP

Create the ZIP with Terraform files at its root:

```bash
cd /Users/johnvngt/Github/scca/3_oci-scca-iam-access-policies-orm
zip -r ../oci-scca-iam-access-policies-orm.zip . \
  -x '.terraform/*' \
  -x 'terraform.tfvars' \
  -x '*.tfplan' \
  -x '.DS_Store'
unzip -l ../oci-scca-iam-access-policies-orm.zip
```

Do not package credentials, AD Bridge configuration, private keys, API keys, or real `terraform.tfvars` values.

## 6. Create the ORM stack

1. In OCI Console, open **Developer Services → Resource Manager → Stacks**.
2. Select **Create stack**, then **My configuration**.
3. Upload `oci-scca-iam-access-policies-orm.zip`.
4. Select the compartment that stores the Resource Manager stack.
5. Choose Terraform version **1.5.x**.
6. Enter the approved values from section 3.
7. Create the stack without selecting **Run apply**.

## 7. Plan, security review, and Apply

1. Create an ORM **Plan** job.
2. Export/review `generated_policy_statements_json` and compare each statement with `policy-matrix.csv`.
3. Confirm every policy principal uses the intended Stack 02 domain display name and AD-synchronized group name.
4. Confirm each workload policy contains the exact approved compartment OCID—not a tenancy-wide scope.
5. Confirm ORM statements allow only `PLAN` and `APPLY`; no statement may allow `DESTROY`.
6. Confirm no policy statement grants IAM administration, keys/secrets management, or `manage all-resources`.
7. Confirm `groups_without_generated_policy_json` lists expected groups without an approved policy.
8. Obtain the required security/IAM approval for the reviewed plan.
9. Run **Apply** only from that approved plan.

## 8. Verify after Apply

1. Verify the ORM job completed successfully.
2. Save `policy_ocids_json` and `generated_policy_statements_json` as ATO/change evidence.
3. Test a representative synchronized account from each approved group, using least-privilege test cases in the assigned compartment only.
4. Test that denied actions remain denied: IAM changes, unassigned workloads, keys/secrets, and ORM destroy jobs.
5. Verify Cloud Guard responders only for approved SECOPS or Incident-Responder workflows.

## 9. Change and rollback management

- Treat each new functional assignment as an access-change request: update the assignment list, run Plan, review exact statements, and Apply after approval.
- To revoke a grant, remove its workload key, Plan, review the policy deletion/update, then Apply.
- Do not manually edit policies managed by this stack; update the stack configuration instead.
- Do not use a broad Terraform destroy as a rollback. Review and remove only the specific policy resources approved for retirement.
