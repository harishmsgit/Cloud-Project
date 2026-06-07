# Challenges

These take you from "I can make a role" to "I can design secure access like a pro." Each one adds a real-world feature you'll meet on the job. Work through them after finishing all eight steps.

---

## Challenge 1 — Scope the Trust to a Single Session Name with a Condition

**Concept:** Trust policies can use `Condition` blocks just like permission policies. You can require that callers tag their session or use a specific session name pattern, making CloudTrail auditing precise.

**Task:**
1. Edit `ReadOnlyS3AssumeRole`'s trust policy to add a condition requiring MFA:
   ```json
   "Condition": { "Bool": { "aws:MultiFactorAuthPresent": "true" } }
   ```
2. Try assuming the role from `lab-user` without MFA → it should be denied.
3. Configure an MFA device for `iam-lab-user` and assume with `--serial-number` and `--token-code`.

**Key question:** Why is requiring MFA *in the trust policy* stronger than requiring it in the permission policy?

---

## Challenge 2 — Permission Boundaries

**Concept:** A permission boundary is a policy that sets the *maximum* permissions an identity can ever have — even if someone attaches `AdministratorAccess` to it. It's how you let teams create their own roles without letting them escalate to admin.

**Task:**
1. Create a managed policy `DevBoundary` allowing only `s3:*` and `logs:*`.
2. Create a role with `AdministratorAccess` attached **and** `DevBoundary` as its permission boundary.
3. Try an admin action (e.g. `ec2:RunInstances`) with the role → denied, despite the admin policy.
4. Try an S3 action → allowed.

**Key question:** Effective permissions = the **intersection** of the identity's policies and its boundary. Why does that make boundaries safe for delegation?

---

## Challenge 3 — Session Policies (shrink permissions at assume time)

**Concept:** When you assume a role you can pass an inline **session policy** that further restricts the session — never expands it. Useful for handing a script *less* than the role allows.

**Task:**
1. Assume `CrossAccountAuditRole` (which has `ReadOnlyAccess`) but pass a session policy that only allows `s3:ListAllMyBuckets`:
   ```bash
   aws sts assume-role --role-arn <arn> --role-session-name scoped \
     --external-id lab-shared-secret-2026 \
     --policy '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":"s3:ListAllMyBuckets","Resource":"*"}]}'
   ```
2. With the resulting creds, `aws s3 ls` works but `aws ec2 describe-instances` is denied — even though the role allows it.

**Key question:** Why can a session policy only *narrow* the role's permissions, never broaden them?

---

## Challenge 4 — Attribute-Based Access Control (ABAC) with Tags

**Concept:** Instead of writing a policy per team, ABAC grants access based on **matching tags**. A single policy like "you may access resources tagged with your own department" scales to any number of teams.

**Task:**
1. Tag `iam-lab-user` with `department=engineering`.
2. Write a policy allowing `s3:*` only where the bucket tag `department` equals `${aws:PrincipalTag/department}`.
3. Create two buckets, tag one `department=engineering` and one `department=finance`.
4. Confirm the user can act on the engineering bucket but not the finance one.

**Key question:** How does ABAC reduce the number of policies you have to maintain as an org grows?

---

## Challenge 5 — Role Chaining

**Concept:** You can assume Role A, then from Role A assume Role B. This is "role chaining." It's powerful but has a catch: chained sessions are capped at **1 hour** and can't be renewed via chaining.

**Task:**
1. Have `lab-user` assume `ReadOnlyS3AssumeRole`.
2. From that session, attempt to assume `CrossAccountAuditRole` (add the necessary trust + permissions).
3. Observe the max session duration drops to 1 hour.

**Key question:** Why does AWS cap chained-role sessions at 1 hour, and when would you avoid chaining?

---

## Challenge 6 — Replace the Lambda Role's Managed Policy with Least Privilege

**Concept:** `AWSLambdaBasicExecutionRole` grants logging to *all* log groups (`Resource: *`). Real least privilege scopes it to the function's own log group.

**Task:**
1. Detach `AWSLambdaBasicExecutionRole` from `LambdaBasicExecutionRole`.
2. Write an inline policy granting `logs:CreateLogStream` and `logs:PutLogEvents` only on `arn:aws:logs:us-east-1:<account>:log-group:/aws/lambda/role-demo:*`.
3. Invoke the function and confirm logging still works.

**Key question:** What's the trade-off between the convenience of broad managed policies and the safety of scoped inline policies?

---

## Challenge 7 — Detect Unused Roles with IAM Access Advisor

**Concept:** AWS tracks when each role was last used and which services it has actually accessed. This is how you find and prune over-permissioned or dead roles.

**Task:**
1. IAM → Roles → pick any role → **Last accessed** tab (or `aws iam get-service-last-accessed-details`).
2. Identify which granted services the role has **never** used.
3. Propose a tighter permission policy based on what's actually used.

**Key question:** How would you build a recurring process to keep role permissions matched to real usage over time?
</content>
