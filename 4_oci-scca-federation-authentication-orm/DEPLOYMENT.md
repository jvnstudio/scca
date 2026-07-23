# Deploy Stack 04: AD FS Federation and Authentication

Deploy this stack with **OCI Resource Manager (ORM)**. It registers AD FS as a SAML identity provider for the Stack 02 identity domain. It does not create accounts, passwords, MFA factors, AD FS trusts, IdP routing rules, or automatic redirection.

## 1. Prerequisites

Before creating the ORM stack, confirm:

- Stack 02 is deployed with `group_provisioning_mode = AD_SYNC`.
- Synchronized pilot users already exist in OCI and have a **unique Primary email address** equal to their AD enterprise email.
- Two institutional OCI-local break-glass accounts exist outside AD FS/AD synchronization, each with independently controlled OCI-native MFA and credentials.
- Both break-glass accounts have been tested in separate private browsers, including a harmless read-only OCI administration action.
- The AD FS team can create a relying-party trust, apply the approved MFA access-control policy, and export current federation metadata.
- An approved change ticket exists for activation and pilot routing.

For privileged AD-Admins, AD FS must require the approved PIV/CAC or YubiKey factor. OCI receives the SAML result; AD FS policy and logs are the evidence of the physical-factor decision.

## 2. Validate and package

Use Terraform 1.5.7:

```bash
cd /Users/johnvngt/Github/scca/4_oci-scca-federation-authentication-orm
tfenv install 1.5.7
tfenv use 1.5.7
terraform init -backend=false
terraform fmt -check -recursive
terraform validate
```

Retain `.terraform.lock.hcl`. Do not commit or package `.terraform/`, real variable files, AD credentials, private keys, tokens, passwords, recovery codes, or MFA seeds.

Export and encode approved AD FS metadata:

```bash
curl --fail --silent --show-error https://<adfs-farm>/FederationMetadata/2007-06/FederationMetadata.xml -o FederationMetadata.xml
base64 < FederationMetadata.xml | tr -d '\n'
```

Review the metadata issuer, SSO endpoints, signing certificate, and expiration before use. Metadata carries public certificates, not private keys.

Build the ZIP with Terraform files at the archive root:

```bash
zip -r ../oci-scca-federation-authentication-orm.zip . \
  -x '.terraform/*' \
  -x 'terraform.tfvars' \
  -x '*.tfplan' \
  -x '.DS_Store'
```

## 3. Pass 1: register disabled and hidden

In ORM, create a stack from the ZIP, select Terraform **1.5.x**, and enter Stack 02’s `identity_domain_url` and `identity_domain_display_name`, the approved AD FS metadata, and these values:

```hcl
group_provisioning_mode      = "AD_SYNC"
activate_adfs_idp            = false
publish_adfs_on_login_page   = false
break_glass_account_count    = 0
```

Run Plan. Confirm it creates exactly one `oci_identity_domains_identity_provider` and that:

- `enabled` and `shown_on_login_page` are false;
- JIT creation, attribute update, and group mapping are false;
- encrypted assertions are required;
- correlation uses SAML `NameID` and `emails[primary eq true].value`.

Apply and record the OCI service-provider metadata URL and AD FS metadata SHA-256 output.

## 4. Configure AD FS and verify correlation

In AD FS, import the OCI service-provider metadata as a relying party. Follow [ADFS_FEDERATION_RUNBOOK.md](ADFS_FEDERATION_RUNBOOK.md) and [adfs-claims-contract.csv](adfs-claims-contract.csv).

Configure AD FS to:

- issue the unique enterprise email as SAML NameID with Email format;
- sign SAML responses with SHA-256;
- encrypt assertions with OCI’s relying-party encryption certificate;
- apply the approved AD-Admins PIV/CAC-or-YubiKey requirement; and
- limit the relying party to the approved pilot population initially.

In OCI, verify the provider’s immutable correlation rule maps SAML NameID to the OCI user’s Primary email. Do not enable JIT or edit the correlation setting manually. If it is incorrect, leave the provider disabled and recreate it through an approved ORM change.

## 5. Pass 2: activate but keep hidden

Only after both break-glass tests and AD FS configuration succeed, update the ORM variables:

```hcl
activate_adfs_idp                     = true
publish_adfs_on_login_page            = false
break_glass_account_count             = 2
break_glass_last_test_date            = "YYYY-MM-DD"
break_glass_verification_confirmation = "I_HAVE_TESTED_TWO_LOCAL_BREAK_GLASS_ACCOUNTS"
activation_change_ticket              = "CHG-123456"
activation_confirmation               = "ACTIVATE_ADFS_IDP_AFTER_SUCCESSFUL_TEST"
```

Run Plan and Apply. The IdP remains hidden; activation alone does not direct users to AD FS.

## 6. Create pilot routing, test, then expand

With a tested OCI-local break-glass session kept open:

1. Manually create or update an OCI IdP policy rule that assigns AD FS to the approved **pilot** population only.
2. Exclude both cloud-local break-glass accounts.
3. Keep local authentication available; do not configure global automatic redirection.
4. Run every case in [federation-test-plan.csv](federation-test-plan.csv) from clean private browsers.
5. Verify AD FS logs prove the required PIV/CAC or YubiKey factor for the AD-Admins test.
6. Confirm unmatched email, duplicate-email, disabled-account, invalid-assertion, and missing-MFA cases fail as expected without creating JIT users.
7. Re-test both local break-glass accounts from separate clean browsers.
8. Expand the routing rule to the approved synchronized enterprise population only after security approval.

## 7. Pass 3: publish

After pilot and break-glass tests pass, update:

```hcl
publish_adfs_on_login_page = true
publication_confirmation   = "PUBLISH_ADFS_AFTER_FEDERATION_AND_BREAK_GLASS_TESTS"
```

Run Plan and Apply. Confirm the only intended change is login-page visibility. Re-run the positive and recovery test cases and retain plans, job logs, metadata hash, AD FS evidence, routing-rule evidence, and break-glass evidence.

## 8. Rollback and operations

If federation fails, use the existing local break-glass session to disable/remove the pilot IdP policy rule first. Then set `publish_adfs_on_login_page = false` and `activate_adfs_idp = false` in ORM, Plan, and Apply. Do not destroy the provider during an incident.

Re-test break-glass access quarterly and after AD FS, AD, DNS, certificate, network, routing, or identity-domain changes. Start AD FS certificate rollover at least 60 days before expiration.
