# IAM & STS CLI Reference (Cheat Sheet)

A quick-reference of the AWS CLI commands used across this project, plus the most common IAM/STS commands you'll reach for in real work. Commands use the project's resource names (`iam-lab-user`, `ReadOnlyS3AssumeRole`, etc.) — swap in your own as needed. Region is **`us-east-1`** throughout.

> New to a command? Add `help` to any command for the full docs, e.g. `aws iam create-role help`.

---

## 1. Exporting Temporary Credentials (Linux, macOS, Windows)

When you run `aws sts assume-role` manually (Step 2.5), it returns temporary credentials you must place in environment variables. **All three are required** — the `AWS_SESSION_TOKEN` is what makes them valid as temporary credentials.

### Linux / macOS (bash, zsh)

```bash
export AWS_ACCESS_KEY_ID="ASIA...."
export AWS_SECRET_ACCESS_KEY="...."
export AWS_SESSION_TOKEN="...."

# clear them when done:
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
```

### Windows — PowerShell

```powershell
$env:AWS_ACCESS_KEY_ID="ASIA...."
$env:AWS_SECRET_ACCESS_KEY="...."
$env:AWS_SESSION_TOKEN="...."

# clear them when done:
Remove-Item Env:AWS_ACCESS_KEY_ID, Env:AWS_SECRET_ACCESS_KEY, Env:AWS_SESSION_TOKEN
```

### Windows — Command Prompt (cmd.exe)

```bat
:: Do NOT wrap values in quotes — cmd keeps the quotes as part of the value
set AWS_ACCESS_KEY_ID=ASIA....
set AWS_SECRET_ACCESS_KEY=....
set AWS_SESSION_TOKEN=....

:: clear them when done (empty value = deletes the variable):
set AWS_ACCESS_KEY_ID=
set AWS_SECRET_ACCESS_KEY=
set AWS_SESSION_TOKEN=
```

> **These env vars override `~/.aws/credentials`.** Forgetting to clear them is the #1 cause of `ExpiredToken` errors later. The `~/.aws/config` profile approach (Step 2.6) refreshes credentials automatically and avoids this.

### One-liners that set all three for you

These call `assume-role`, parse the JSON, and export in one step — no copy/paste.

**Linux / macOS (needs `jq`):**

```bash
CREDS=$(aws sts assume-role \
  --role-arn arn:aws:iam::111122223333:role/ReadOnlyS3AssumeRole \
  --role-session-name cli-session --query Credentials --output json)
export AWS_ACCESS_KEY_ID=$(echo "$CREDS" | jq -r .AccessKeyId)
export AWS_SECRET_ACCESS_KEY=$(echo "$CREDS" | jq -r .SecretAccessKey)
export AWS_SESSION_TOKEN=$(echo "$CREDS" | jq -r .SessionToken)
```

**Windows — PowerShell:**

```powershell
$c = (aws sts assume-role `
  --role-arn arn:aws:iam::111122223333:role/ReadOnlyS3AssumeRole `
  --role-session-name cli-session --query Credentials | ConvertFrom-Json)
$env:AWS_ACCESS_KEY_ID = $c.AccessKeyId
$env:AWS_SECRET_ACCESS_KEY = $c.SecretAccessKey
$env:AWS_SESSION_TOKEN = $c.SessionToken
```

> **Better alternative — skip env vars entirely.** Define a profile with `role_arn` + `source_profile` (Step 2.6) and just pass `--profile lab-readonly`. The CLI assumes the role and refreshes credentials for you.

---

## 2. Configuration & Profiles

| Goal | Command |
|------|---------|
| Configure default credentials | `aws configure` |
| Configure a named profile | `aws configure --profile lab-user` |
| Use a profile for one command | `aws s3 ls --profile lab-readonly` |
| Set a profile for the whole session (Linux/macOS) | `export AWS_PROFILE=lab-readonly` |
| Set a profile for the whole session (PowerShell) | `$env:AWS_PROFILE="lab-readonly"` |
| List configured profiles | `aws configure list-profiles` |
| Show current effective config | `aws configure list` |

A role-assuming profile in `~/.aws/config`:

```ini
[profile lab-readonly]
role_arn = arn:aws:iam::111122223333:role/ReadOnlyS3AssumeRole
source_profile = lab-user
region = us-east-1
# external_id = lab-shared-secret-2026   # for cross-account roles (Step 5)
```

---

## 3. STS Commands (who am I, and assuming roles)

| Goal | Command |
|------|---------|
| Show your current identity (account, ARN, user/role) | `aws sts get-caller-identity` |
| Assume a role | `aws sts assume-role --role-arn <arn> --role-session-name s1` |
| Assume with an External ID (cross-account, Step 5) | `aws sts assume-role --role-arn <arn> --role-session-name s1 --external-id lab-shared-secret-2026` |
| Assume with a shorter session (15 min) | `aws sts assume-role --role-arn <arn> --role-session-name s1 --duration-seconds 900` |
| Assume with a scoped-down session policy | `aws sts assume-role --role-arn <arn> --role-session-name s1 --policy '<json>'` |
| Federated assume (OIDC/web identity, Step 7) | `aws sts assume-role-with-web-identity --role-arn <arn> --role-session-name s1 --web-identity-token <jwt>` |
| Get a session token (MFA scenarios) | `aws sts get-session-token --serial-number <mfa-arn> --token-code 123456` |

---

## 4. IAM Users

| Goal | Command |
|------|---------|
| Create a user | `aws iam create-user --user-name iam-lab-user` |
| List users | `aws iam list-users` |
| Get one user | `aws iam get-user --user-name iam-lab-user` |
| Create access keys | `aws iam create-access-key --user-name iam-lab-user` |
| List a user's access keys | `aws iam list-access-keys --user-name iam-lab-user` |
| Delete an access key | `aws iam delete-access-key --user-name iam-lab-user --access-key-id AKIA...` |
| Attach an inline policy to a user | `aws iam put-user-policy --user-name iam-lab-user --policy-name P --policy-document file://p.json` |
| List a user's inline policies | `aws iam list-user-policies --user-name iam-lab-user` |
| Create console login | `aws iam create-login-profile --user-name iam-lab-user --password '<pw>'` |
| Delete console login | `aws iam delete-login-profile --user-name iam-lab-user` |
| Delete the user | `aws iam delete-user --user-name iam-lab-user` |

---

## 5. IAM Roles

| Goal | Command |
|------|---------|
| Create a role (with trust policy) | `aws iam create-role --role-name R --assume-role-policy-document file://trust.json` |
| List roles | `aws iam list-roles` |
| Get one role (shows trust policy) | `aws iam get-role --role-name R` |
| View only the trust policy | `aws iam get-role --role-name R --query "Role.AssumeRolePolicyDocument"` |
| Update the trust policy | `aws iam update-assume-role-policy --role-name R --policy-document file://trust.json` |
| Attach a managed permission policy | `aws iam attach-role-policy --role-name R --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess` |
| Add an inline permission policy | `aws iam put-role-policy --role-name R --policy-name P --policy-document file://p.json` |
| List attached (managed) policies | `aws iam list-attached-role-policies --role-name R` |
| List inline policies | `aws iam list-role-policies --role-name R` |
| Detach a managed policy | `aws iam detach-role-policy --role-name R --policy-arn <arn>` |
| Delete an inline policy | `aws iam delete-role-policy --role-name R --policy-name P` |
| Delete the role | `aws iam delete-role --role-name R` |

> A role won't delete (`DeleteConflict`) until you detach/delete **all** its policies. List both managed and inline first.

---

## 6. IAM Policies (standalone / customer-managed)

| Goal | Command |
|------|---------|
| Create a standalone managed policy | `aws iam create-policy --policy-name P --policy-document file://p.json` |
| List your customer-managed policies | `aws iam list-policies --scope Local` |
| Get a policy (metadata) | `aws iam get-policy --policy-arn <arn>` |
| View a policy's current JSON | `aws iam get-policy-version --policy-arn <arn> --version-id v1` |
| Delete a policy | `aws iam delete-policy --policy-arn <arn>` |

---

## 7. Instance Profiles (EC2, Step 4)

| Goal | Command |
|------|---------|
| Create an instance profile | `aws iam create-instance-profile --instance-profile-name EC2S3AccessProfile` |
| Put a role inside it | `aws iam add-role-to-instance-profile --instance-profile-name EC2S3AccessProfile --role-name EC2S3AccessRole` |
| Inspect it (shows the role inside) | `aws iam get-instance-profile --instance-profile-name EC2S3AccessProfile` |
| Remove the role from it | `aws iam remove-role-from-instance-profile --instance-profile-name EC2S3AccessProfile --role-name EC2S3AccessRole` |
| Delete the instance profile | `aws iam delete-instance-profile --instance-profile-name EC2S3AccessProfile` |

---

## 8. OIDC Identity Providers (GitHub Actions, Step 7)

| Goal | Command |
|------|---------|
| Create an OIDC provider | `aws iam create-open-id-connect-provider --url https://token.actions.githubusercontent.com --client-id-list sts.amazonaws.com` |
| List OIDC providers | `aws iam list-open-id-connect-providers` |
| Get one provider | `aws iam get-open-id-connect-provider --open-id-connect-provider-arn <arn>` |
| Delete a provider | `aws iam delete-open-id-connect-provider --open-id-connect-provider-arn <arn>` |

---

## 9. Inspection & Troubleshooting

These help you answer "why is this denied?" and audit access.

| Goal | Command |
|------|---------|
| Who am I right now? | `aws sts get-caller-identity` |
| Dry-run: would this action be allowed? | `aws iam simulate-principal-policy --policy-source-arn <user-or-role-arn> --action-names s3:GetObject --resource-arns <arn>` |
| When did a role last use each service? (Access Advisor) | `aws iam generate-service-last-accessed-details --arn <role-arn>` then `aws iam get-service-last-accessed-details --job-id <id>` |
| Full account IAM dump (users/roles/policies) | `aws iam get-account-authorization-details` |
| List entities a policy is attached to | `aws iam list-entities-for-policy --policy-arn <arn>` |
| Validate JSON before submitting (local) | `python -m json.tool < trust.json` |

---

## 10. Cleanup Quick Commands

The full, ordered teardown is in [Step 8 — Cleanup](./steps/08-cleanup.md). Reminder of the dependency order:

```
detach/delete role policies  ─►  delete role
remove role from profile     ─►  delete instance profile
delete user keys + login + inline policies  ─►  delete user
```

---

## See Also

- [Step 2 — Assuming a role from the CLI](./steps/02-role-assumed-by-user-cli.md) — the export commands in context
- [troubleshooting.md](./troubleshooting.md) — `AccessDenied`, `ExpiredToken`, `MalformedPolicyDocument`, and more
- [Official AWS CLI IAM reference](https://docs.aws.amazon.com/cli/latest/reference/iam/) and [STS reference](https://docs.aws.amazon.com/cli/latest/reference/sts/)
</content>
