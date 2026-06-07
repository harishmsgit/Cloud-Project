# Step 8 — Cleanup: Delete Every Role, Policy, and User

## Why This Matters

IAM resources don't cost money, but leaving them around is poor hygiene: stale roles widen your attack surface and clutter the account. The **one** thing that *does* cost money is any EC2 instance from Step 4 — kill that first.

Delete in dependency order: detach/delete policies *before* the role, remove a role from its instance profile *before* deleting the profile, and remove access keys *before* deleting the user.

---

## Step 8.1 — Terminate the EC2 Instance (if you launched one)

> This is the only billable resource in the project. Do it first.

**Console:** EC2 → **Instances** → select the lab instance → **Instance state** → **Terminate instance**.

**CLI:**

```bash
aws ec2 describe-instances \
  --filters "Name=iam-instance-profile.arn,Values=*EC2S3AccessProfile*" \
  --query "Reservations[].Instances[].InstanceId" --output text

aws ec2 terminate-instances --instance-ids <instance-id-from-above>
```

---

## Step 8.2 — Delete the Lambda Function (if created in Step 3)

```bash
aws lambda delete-function --function-name role-demo
aws logs delete-log-group --log-group-name /aws/lambda/role-demo
```

---

## Step 8.3 — Delete the Roles

Each role must have its policies detached/deleted first. The snippets below handle both managed (`detach-role-policy`) and inline (`delete-role-policy`) cases.

```bash
# Step 2 — ReadOnlyS3AssumeRole (managed policy attached)
aws iam detach-role-policy --role-name ReadOnlyS3AssumeRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam delete-role --role-name ReadOnlyS3AssumeRole

# Step 3 — LambdaBasicExecutionRole
aws iam detach-role-policy --role-name LambdaBasicExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name LambdaBasicExecutionRole

# Step 4 — EC2S3AccessRole (must leave the instance profile first)
aws iam remove-role-from-instance-profile \
  --instance-profile-name EC2S3AccessProfile --role-name EC2S3AccessRole
aws iam delete-instance-profile --instance-profile-name EC2S3AccessProfile
aws iam detach-role-policy --role-name EC2S3AccessRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam delete-role --role-name EC2S3AccessRole

# Step 5 — CrossAccountAuditRole
aws iam detach-role-policy --role-name CrossAccountAuditRole \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
aws iam delete-role --role-name CrossAccountAuditRole

# Step 6 — ECS roles
aws iam detach-role-policy --role-name ECSAppTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
aws iam delete-role --role-name ECSAppTaskExecutionRole

aws iam delete-role-policy --role-name ECSAppTaskRole --policy-name AppS3Read
aws iam delete-role --role-name ECSAppTaskRole

# Step 7 — GitHubActionsDeployRole
aws iam delete-role-policy --role-name GitHubActionsDeployRole --policy-name EcrPush
aws iam delete-role --role-name GitHubActionsDeployRole
```

> **If `delete-role` fails with `DeleteConflict`,** the role still has a policy attached. List what's left with `aws iam list-attached-role-policies --role-name <name>` and `aws iam list-role-policies --role-name <name>`, remove them, then retry.

---

## Step 8.4 — Delete the OIDC Identity Provider (Step 7)

```bash
aws iam list-open-id-connect-providers   # copy the ARN

aws iam delete-open-id-connect-provider \
  --open-id-connect-provider-arn arn:aws:iam::111122223333:oidc-provider/token.actions.githubusercontent.com
```

---

## Step 8.5 — Delete the Practice User

Remove its inline policy and access keys before deleting the user.

```bash
# Inline policy from Step 2
aws iam delete-user-policy --user-name iam-lab-user --policy-name AssumeReadOnlyS3

# Access keys from Step 1.3
aws iam list-access-keys --user-name iam-lab-user   # copy each AccessKeyId
aws iam delete-access-key --user-name iam-lab-user --access-key-id <AccessKeyId>

# Console login profile (if you enabled console access)
aws iam delete-login-profile --user-name iam-lab-user

aws iam delete-user --user-name iam-lab-user
```

---

## Step 8.6 — Clean Up Local Files and Profiles

```bash
# Delete the JSON policy files you created
rm -f trust-policy-*.json user-can-assume.json task-role-perms.json oidc-perms.json \
      handler.py function.zip response.json
```

Then edit `~/.aws/config` and `~/.aws/credentials` and **remove the lab profiles** you added:
`[profile lab-user]`, `[profile lab-readonly]`, `[profile cross-audit]`, and the `[lab-user]` credentials block.

---

## Step 8.7 — (If You Created Any) Delete the Test S3 Bucket

If you made a bucket to test reads against:

```bash
aws s3 rb s3://my-app-bucket --force
```

---

## Verification

Run these — each list should **no longer** contain the lab resources:

```bash
aws iam list-roles --query "Roles[?contains(RoleName, 'AssumeRole') || contains(RoleName, 'ECSApp') || contains(RoleName, 'GitHubActions') || contains(RoleName, 'EC2S3') || contains(RoleName, 'CrossAccount') || contains(RoleName, 'LambdaBasic')].RoleName"
aws iam list-users --query "Users[?UserName=='iam-lab-user'].UserName"
aws iam list-open-id-connect-providers
aws iam list-instance-profiles --query "InstanceProfiles[?InstanceProfileName=='EC2S3AccessProfile']"
```

All should return empty.

---

## Cleanup Checklist

- [ ] EC2 instance terminated (Step 4)
- [ ] Lambda function + log group deleted (Step 3)
- [ ] All six roles deleted
- [ ] Instance profile deleted
- [ ] OIDC provider deleted
- [ ] `iam-lab-user` access keys, login profile, inline policy, and user deleted
- [ ] Local JSON files removed and `~/.aws` lab profiles cleaned out
- [ ] Test S3 bucket emptied and deleted (if created)

You're done — the account is back to where it started.
</content>
