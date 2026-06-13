# Step 3 — IAM: Instance Profile + GitHub OIDC Deploy Role

This project uses **two roles with two completely different trust models**, and seeing them
side by side is the whole IAM lesson:

| | Role 1 — Instance profile | Role 2 — GitHub deploy role |
|--|---------------------------|------------------------------|
| **Name** | `WebAppInstanceRole` | `GitHubActionsDeployRole` |
| **Who assumes it** | The EC2 **service**, on the instance's behalf | A **GitHub Actions** job, from outside AWS |
| **Trust type** | AWS **service** principal | **Web identity federation** (OIDC) |
| **Trusts** | `ec2.amazonaws.com` | `token.actions.githubusercontent.com` |
| **Assume action** | `sts:AssumeRole` | `sts:AssumeRoleWithWebIdentity` |
| **Used in** | Steps 4, 7, 10 (runtime) | Step 10 (CI/CD deploy) |

A **trust policy** (the "trust relationship") answers one question: *who is allowed to
become this role?* The **permission policy** answers a different one: *what can the role do
once assumed?* Keep them separate in your head.

---

## 3.2 Role 1 — The EC2 Instance Profile

EC2 instances need to let SSM manage them (deploys + shell) and let the CloudWatch agent
push metrics and logs. We grant these through an **instance profile** — an IAM role EC2
assumes on the instance's behalf, so no credentials are ever stored on the box.

**Role name:** `WebAppInstanceRole` · **Trusts:** `ec2.amazonaws.com`

| Permission set | Managed policy | Why It's Needed |
|----------------|----------------|-----------------|
| SSM management | `AmazonSSMManagedInstanceCore` | SSM Run Command deploys (Step 10) + no-SSH shell |
| CloudWatch agent | `CloudWatchAgentServerPolicy` | Push memory/disk metrics and ship logs (Step 7) |

Why these two and nothing more? Least privilege. The instance never needs to launch other
instances, so we don't grant that.

### Console

1. **IAM console → Roles → Create role**.
2. **Trusted entity type:** AWS service → **EC2** → Next.
3. Attach: `AmazonSSMManagedInstanceCore` and `CloudWatchAgentServerPolicy`.
4. Next → **Role name:** `WebAppInstanceRole` → **Create role**.

> An **instance profile** is just a container that hands the role to the OS. The Console
> creates the matching instance profile automatically; the CLI does it explicitly (3.6).

---

## 3.3 The GitHub OIDC Identity Provider (create once per account)

Before any GitHub repo can assume an AWS role, AWS has to be told to **trust GitHub's
token issuer**. That's what an **IAM OIDC identity provider** is: a one-time registration
that says "I trust tokens signed by `token.actions.githubusercontent.com`."

1. **IAM → Identity providers → Add provider**.

   | Field | Value |
   |-------|-------|
   | Provider type | **OpenID Connect** |
   | Provider URL | `https://token.actions.githubusercontent.com` |
   | Audience | `sts.amazonaws.com` |

2. **Add provider.**

This is **account-wide** — if you also build the serverless companion project, it reuses
the same provider. Without it, the role in 3.4 can't reference a federated principal.

---

## 3.4 Role 2 — The GitHub OIDC Deploy Role

This role lets a GitHub Actions workflow get **short-lived** AWS credentials by proving its
identity with an OIDC token — **no long-lived `AWS_ACCESS_KEY_ID` stored in GitHub**. The
trust policy is the security boundary, so we'll dissect it line by line in 3.5.

**Trust policy** (replace `<ACCOUNT_ID>` and `ORG/REPO`):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:ORG/REPO:ref:refs/heads/main"
      }
    }
  }]
}
```

**Console:** IAM → Roles → Create role → **Web identity** → Identity provider
`token.actions.githubusercontent.com`, Audience `sts.amazonaws.com` → add the `sub`
condition on the next screen (or paste the JSON above via **Edit trust policy**) → name it
`GitHubActionsDeployRole`.

---

## 3.5 Anatomy of the OIDC Trust Relationship

Every field in that trust policy is doing a job. This is the part to understand deeply —
get the `Condition` wrong and *any* GitHub repo on the internet could assume your role.

| Element | Value here | What it does / why it matters |
|---------|-----------|-------------------------------|
| `Version` | `2012-10-17` | The IAM policy language version. Always this exact date — it's not "today", it's the schema version. |
| `Effect` | `Allow` | This statement **grants** the ability to assume. (Trust policies are almost always `Allow`.) |
| `Principal.Federated` | the OIDC provider ARN | **Who** is trusted: the identity provider you registered in 3.3. It points at GitHub's token issuer, not at a specific repo — the narrowing happens in `Condition`. |
| `Action` | `sts:AssumeRoleWithWebIdentity` | The **only** STS call a federated web identity can use. (Service roles use `sts:AssumeRole`; this is its OIDC cousin.) Without it, the token is useless. |
| `Condition` | the two claim checks below | **The real security boundary.** `Principal` trusts all of GitHub; `Condition` narrows it to *your* repo and branch. Omitting it = every GitHub Actions run on Earth can assume the role. |

### The two claim checks inside `Condition`

OIDC tokens carry **claims** (key/value facts about the request). We assert on two:

| Claim check | Operator | Meaning |
|-------------|----------|---------|
| `...:aud` = `sts.amazonaws.com` | `StringEquals` | **Audience** — the token must be minted *for AWS STS*, not for some other service. This must match **exactly**, so we use `StringEquals`. It blocks a token issued for a different audience from being replayed against AWS. |
| `...:sub` = `repo:ORG/REPO:ref:refs/heads/main` | `StringLike` | **Subject** — *which* GitHub workload is calling. The format is `repo:<org>/<repo>:ref:refs/heads/<branch>`. This is what pins the role to **your** repository and the `main` branch. |

**Why `StringLike` for `sub` (and not `StringEquals`)?** `StringLike` supports the `*`
wildcard, which you'll want once your `sub` needs flexibility — e.g.
`repo:ORG/REPO:ref:refs/heads/*` (any branch) or `repo:ORG/REPO:*` (any ref, including
tags and pull requests). For a single fixed branch you *could* use `StringEquals`; we use
`StringLike` so widening later is a one-character edit, not an operator change.

### The `sub` claim — common shapes

| `sub` pattern | Who can assume |
|---------------|----------------|
| `repo:ORG/REPO:ref:refs/heads/main` | Only the `main` branch of one repo (what we use) |
| `repo:ORG/REPO:ref:refs/heads/*` | Any branch of that repo |
| `repo:ORG/REPO:environment:production` | Only jobs targeting the `production` GitHub Environment |
| `repo:ORG/REPO:pull_request` | Only pull-request-triggered runs |
| `repo:ORG/*` | ⚠️ Any repo in the org — usually too broad |

> ⚠️ **The classic mistake:** a `Principal` of GitHub's provider with **no `sub` condition**
> (or a `sub` of `repo:*`). That trusts the entire planet's GitHub Actions. Always pin
> `sub` to your org/repo, and scope it to a branch or environment for production.

---

## 3.6 Permission Policy for the Deploy Role (least privilege)

Trust says *who*; this says *what*. The deploy role needs only enough to push the artifact
and trigger the rollout (used in Step 10):

| Permission | Why It's Needed |
|------------|-----------------|
| `s3:PutObject`, `s3:ListBucket` on the deploy bucket | Upload the new app artifact |
| `ssm:SendCommand` | Trigger the rollout on the instances |
| `ssm:GetCommandInvocation`, `ssm:ListCommandInvocations` | Poll the deploy result |

Scope every `Resource` to the specific bucket ARN and SSM document — never `"Resource": "*"`.
(The full JSON is challenge 5 in [challenges.md](../challenges.md).)

---

## 3.7 AWS CLI (Alternative — Both Roles)

```bash
REGION=us-east-1
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# ---- Role 1: EC2 instance profile ----
cat > ec2-trust.json <<'JSON'
{ "Version": "2012-10-17",
  "Statement": [{ "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole" }] }
JSON

aws iam create-role --role-name WebAppInstanceRole \
  --assume-role-policy-document file://ec2-trust.json
aws iam attach-role-policy --role-name WebAppInstanceRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam attach-role-policy --role-name WebAppInstanceRole \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
aws iam create-instance-profile --instance-profile-name WebAppInstanceRole
aws iam add-role-to-instance-profile \
  --instance-profile-name WebAppInstanceRole --role-name WebAppInstanceRole

# ---- OIDC provider (once per account) ----
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com

# ---- Role 2: GitHub OIDC deploy role ----
# Replace ORG/REPO before running.
cat > github-trust.json <<JSON
{ "Version": "2012-10-17",
  "Statement": [{ "Effect": "Allow",
    "Principal": {"Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/token.actions.githubusercontent.com"},
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {"token.actions.githubusercontent.com:aud": "sts.amazonaws.com"},
      "StringLike": {"token.actions.githubusercontent.com:sub": "repo:ORG/REPO:ref:refs/heads/main"}
    } }] }
JSON

aws iam create-role --role-name GitHubActionsDeployRole \
  --assume-role-policy-document file://github-trust.json
# Permission policy (S3 + SSM, scoped) is attached in Step 10 — see challenges.md for JSON.
```

> Modern AWS no longer requires the provider **thumbprint** for the GitHub issuer — STS
> validates it against a trusted CA — so `create-open-id-connect-provider` needs only the
> URL and the `sts.amazonaws.com` client id.

---

## Checkpoint

- [ ] Role `WebAppInstanceRole` trusts `ec2.amazonaws.com`; has SSM + CloudWatch agent policies
- [ ] An instance profile named `WebAppInstanceRole` exists
- [ ] OIDC identity provider `token.actions.githubusercontent.com` exists (audience `sts.amazonaws.com`)
- [ ] Role `GitHubActionsDeployRole` trusts the OIDC provider via `sts:AssumeRoleWithWebIdentity`
- [ ] Its trust `Condition` pins `aud` (`StringEquals`) **and** `sub` to `repo:ORG/REPO` (`StringLike`)
- [ ] You can explain why the `sub` condition is the real security boundary

---

**Next:** [Step 4 — Launch EC2 with User Data](./04-launch-ec2.md)
