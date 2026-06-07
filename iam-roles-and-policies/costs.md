# Cost Guide — IAM Roles & Policies

This project is **almost entirely free**. IAM, STS, roles, policies, and the OIDC provider have **no charge at all** — ever, on any account. The only way to spend money here is the *optional* EC2 verification in Step 4.

All prices are for **us-east-1**.

---

## Services Used and Their Pricing Model

| Service | Pricing Model | Free Tier |
|---------|--------------|-----------|
| AWS IAM (users, roles, policies) | **Always free** | N/A — never charged |
| AWS STS (`AssumeRole`, `AssumeRoleWithWebIdentity`) | **Always free** | N/A — never charged |
| IAM OIDC identity provider | **Always free** | N/A — never charged |
| AWS Lambda (Step 3, optional) | Per request + per GB-second | 1M requests + 400,000 GB-s / month |
| Amazon EC2 (Step 4, optional) | Per instance-hour | 750 hrs/month `t2.micro`/`t3.micro` (new accounts, 12 mo) |
| Amazon S3 (test object) | Per GB stored + per request | 5 GB storage / 12 mo (new accounts) |
| Amazon CloudWatch Logs | Per GB ingested + stored | 5 GB ingestion + 5 GB storage / month |

---

## Service-by-Service Breakdown

### 1. IAM, STS, OIDC — $0.00, guaranteed

There is **no charge** to:
- Create users, roles, groups, or policies
- Call `sts:AssumeRole` or `sts:AssumeRoleWithWebIdentity` (any number of times)
- Register or use an OIDC identity provider

You can complete **Steps 1, 2, 5, 6, and 7 in full at $0.00.** These are pure IAM modelling.

---

### 2. AWS Lambda (Step 3, optional verification)

Only incurred if you do the optional "prove it works" function.

| Action | Price | This Project |
|--------|-------|--------------|
| Requests | $0.20 per 1M | A handful of invokes — free |
| Compute | $0.0000166667 per GB-s | Milliseconds at 128 MB — free |

**Cost: $0.00** — far within the 1M-request free tier (which never expires).

---

### 3. Amazon EC2 (Step 4, optional verification) — the only real cost

This is the **single billable resource** in the project, and only if you actually launch an instance to test the instance profile.

| Instance | On-Demand Price | Free Tier |
|----------|-----------------|-----------|
| `t3.micro` | ~$0.0104/hr | 750 hrs/month for 12 months (new accounts) |
| `t2.micro` | ~$0.0116/hr | 750 hrs/month for 12 months (new accounts) |

**If you're outside the free tier**, a forgotten instance costs:

| Left Running | Cost (`t3.micro`) |
|--------------|-------------------|
| 1 hour | ~$0.01 |
| 1 day ⚠️ | ~$0.25 |
| 7 days ⚠️ | ~$1.75 |
| 30 days ⚠️ | ~$7.49 |

> **Terminate the instance the moment you finish Step 4** — or skip Step 4.3 entirely and keep the project at $0.00.

---

### 4. S3 and CloudWatch Logs

| Resource | This Project | Cost |
|----------|--------------|------|
| One test object in S3 | A few KB | ~$0.00 |
| Lambda log group | A few KB | ~$0.00 (within 5 GB free tier) |

---

## Cost by Scenario

| What You Do | Billable Resources | Total |
|-------------|--------------------|-------|
| Steps 1, 2, 5, 6, 7 only (pure IAM) | None | **$0.00** |
| + Step 3 Lambda verification | Lambda (free tier) | **$0.00** |
| + Step 4 EC2 verification, terminated within 1 hr | EC2 (~1 hr) | **~$0.01** (or $0.00 in free tier) |
| ⚠️ Step 4 EC2 **left running** 30 days | EC2 | **~$7.49** |

---

## What You Will NOT Be Charged For

- Any IAM user, role, group, or policy
- Any number of `AssumeRole` / STS calls
- The OIDC identity provider
- Instance profiles
- Trust policy / permission policy evaluation
- AWS Management Console usage

---

## Cleanup Reminder

The only resource that costs money if forgotten is the **EC2 instance from Step 4**. Everything else is free, but should still be removed for hygiene. Run [Step 8 — Cleanup](./steps/08-cleanup.md) to delete it all.
</content>
