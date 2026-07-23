# Deploy the OCI SCCA-Style Compartment Stack

This runbook deploys the compartment-only SCCA-style hierarchy with OCI Resource Manager. It creates no IAM policies, networks, buckets, logging services, or other resources.

## 1. Before you start

You need:

- An OCI tenancy OCID.
- A parent compartment OCID, or a decision to place the SCCA home compartment directly under the tenancy root.
- An OCI region (for example, `us-ashburn-1`) and its naming key (for example, `IAD`).
- A compartment in which to store the Resource Manager stack. This can be different from the hierarchy's parent compartment.
- Permission for the Resource Manager job to manage compartments in the intended parent scope.

Do not put API keys, private keys, user OCIDs, fingerprints, passwords, or other secrets in Terraform variables.

Before applying, check that none of the rendered names are already managed by another Terraform stack. With `enable_compartment_delete = false`, the OCI provider can adopt a same-named existing compartment into this stack's state and update its description.

## 2. Verify the package locally

OCI Resource Manager supports Terraform 1.5.x. Use Terraform 1.5.7 for the local check.

```bash
cd /Users/johnvngt/Github/scca/1_oci-scca-compartments-orm
tfenv install 1.5.7
tfenv use 1.5.7
terraform version
terraform init -backend=false
terraform fmt -check -recursive
terraform validate
```

Expected result: `Success! The configuration is valid.` The generated `.terraform.lock.hcl` locks the tested OCI provider version and should be retained with the configuration.

## 3. Choose and review values

Copy the example values for local reference only:

```bash
cp terraform.tfvars.example terraform.tfvars
```

Set these values:

```hcl
tenancy_ocid            = "ocid1.tenancy..."
parent_compartment_ocid = "" # Empty means tenancy root.
region                  = "us-ashburn-1"
region_key              = "IAD"
resource_label          = "PROD"
workload_postfixes      = ["SWR1", "SWR2"]

enable_logging_compartment = true
enable_backup_compartment  = true
enable_compartment_delete  = false
```

With the defaults, those inputs render this hierarchy:

```text
Selected parent (or tenancy root)
└── OCI-SCCA-LZ-Home-IAD-PROD
    ├── OCI-SCCA-LZ-VDMS-IAD-PROD
    ├── OCI-SCCA-LZ-VDSS-IAD-PROD
    ├── OCI-SCCA-LZ-Logging-IAD-PROD              (when enabled)
    ├── OCI-SCCA-LZ-IAC-TF-Configbackup-PROD     (when enabled)
    ├── OCI-SCCA-LZ-WRK-IAD-SWR1
    └── OCI-SCCA-LZ-WRK-IAD-SWR2
```

`parent_compartment_ocid` identifies the parent of the new Home compartment. It is not the OCI Resource Manager stack-storage compartment.

## 4. Build the upload ZIP

Create the archive with the Terraform files at the ZIP root (not inside an extra directory):

```bash
cd /Users/johnvngt/Github/scca/1_oci-scca-compartments-orm
zip -r ../oci-scca-compartments-orm.zip . \
  -x '.terraform/*' \
  -x 'terraform.tfvars' \
  -x '*.tfplan' \
  -x '.DS_Store'
unzip -l ../oci-scca-compartments-orm.zip
```

Include `.terraform.lock.hcl` if present. Do not include real variable files or secrets.

## 5. Create the Resource Manager stack

1. In OCI Console, open **Developer Services → Resource Manager → Stacks**.
2. Select **Create stack**, then **My configuration**.
3. Upload `oci-scca-compartments-orm.zip`.
4. Choose the compartment that stores the Resource Manager stack.
5. Select Terraform version **1.5.x**.
6. Enter the inputs from section 3 in the variables form.
7. Keep **Allow Terraform to Delete Compartments** disabled.
8. Create the stack without selecting **Run apply**.

## 6. Plan before applying

1. Open the new stack and create a **Plan** job.
2. Read the plan and confirm every action is an `oci_identity_compartment` resource.
3. Confirm the planned parent OCID and every rendered name.
4. Stop if a plan indicates an existing compartment is being adopted, its description is changing unexpectedly, or the hierarchy is in the wrong scope.

Only when the Plan is correct, create an **Apply** job using that reviewed plan.

## 7. Verify after apply

1. Confirm the job completed successfully.
2. Review the stack outputs for the Home, VDMS, VDSS, optional Logging/Backup, and workload compartment OCIDs.
3. In OCI Identity & Security → Compartments, verify the parent/child hierarchy.
4. Record the stack OCID, Apply job OCID, provider lock file version, and outputs in the deployment record.

## 8. Change and deletion safety

- Add a workload by adding a new unique entry to `workload_postfixes`, then run Plan and Apply.
- Treat a rename, a removed workload postfix, or disabling Logging/Backup as an offboarding action. With `enable_compartment_delete = false`, Terraform does **not** delete the compartment; it becomes unmanaged when removed from configuration.
- Keep `enable_compartment_delete = false` for normal operations.
- Enable deletion only through a separately reviewed change, after verifying every affected compartment is empty and no longer needed. Run a Plan before Apply.

For normal changes, always update the stack configuration, run Plan, review it, and then Apply the reviewed plan.
