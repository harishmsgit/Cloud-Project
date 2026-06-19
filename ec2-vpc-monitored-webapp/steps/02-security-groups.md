# Step 2 — Security Groups: The Virtual Firewalls

Security groups are stateful firewalls attached to the ALB and the EC2 instances. We
create two so traffic can only flow one way: **Internet → ALB → EC2**, never
**Internet → EC2** directly.

---

## 2.1 What You're Creating

| Security Group | Attached to | Inbound rule | Why |
|----------------|-------------|--------------|-----|
| `alb-sg` | The ALB | TCP **80** from `0.0.0.0/0` | The public reaches the load balancer over HTTP |
| `ec2-sg` | The EC2 instances | TCP **5000** from **`alb-sg`** | Only the ALB may reach the app port — not the internet |

**The key idea:** `ec2-sg`'s source is **another security group** (`alb-sg`), not a CIDR.
That means "allow traffic from anything wearing the `alb-sg` badge" — i.e. only the load
balancer. Even though the instances have no public IP, this guarantees the app port can't
be reached from anywhere else in the VPC either.

> We do **not** open port 22 (SSH). Step 10 uses **SSM Session Manager** for shell access,
> so there's no need for an inbound SSH rule at all — a big security win.

---

## 2.2 Console — Create `alb-sg`

1. **EC2 console → Security Groups → Create security group**.

   | Field | Value |
   |-------|-------|
   | Name | `alb-sg` |
   | Description | `ALB ingress from internet` |
   | VPC | `webapp-vpc` |

2. **Inbound rules → Add rule**:

   | Type | Port | Source |
   |------|------|--------|
   | HTTP | 80 | `0.0.0.0/0` |

3. Leave outbound as default (all). **Create security group**.

---

## 2.3 Console — Create `ec2-sg`

1. **Create security group**.

   | Field | Value |
   |-------|-------|
   | Name | `ec2-sg` |
   | Description | `App port from ALB only` |
   | VPC | `webapp-vpc` |

2. **Inbound rules → Add rule**:

   | Type | Port | Source |
   |------|------|--------|
   | Custom TCP | 5000 | **Custom →** select `alb-sg` |

3. Leave outbound as default (all — instances need outbound for updates via NAT).
   **Create security group**.

---

## 2.4 AWS CLI (Alternative)

```bash
REGION=ap-south-1
# VPC_ID from Step 1

ALB_SG=$(aws ec2 create-security-group --group-name alb-sg \
  --description "ALB ingress from internet" --vpc-id $VPC_ID \
  --query 'GroupId' --output text --region $REGION)
aws ec2 authorize-security-group-ingress --group-id $ALB_SG \
  --protocol tcp --port 80 --cidr 0.0.0.0/0 --region $REGION

EC2_SG=$(aws ec2 create-security-group --group-name ec2-sg \
  --description "App port from ALB only" --vpc-id $VPC_ID \
  --query 'GroupId' --output text --region $REGION)
aws ec2 authorize-security-group-ingress --group-id $EC2_SG \
  --protocol tcp --port 5000 --source-group $ALB_SG --region $REGION

echo "ALB_SG=$ALB_SG  EC2_SG=$EC2_SG"

##Actual value.

VPC_ID=vpc-06bbc38be0501663f

# ALB Security Group
ALB_SG=$(aws ec2 create-security-group \
  --group-name alb-sg \
  --description "ALB ingress from internet" \
  --vpc-id $VPC_ID \
  --query 'GroupId' --output text --region $REGION)


# EC2 Security Group
EC2_SG=$(aws ec2 create-security-group \
  --group-name ec2-sg \
  --description "App port from ALB only" \
  --vpc-id $VPC_ID \
  --query 'GroupId' --output text --region $REGION

aws ec2 describe-security-groups \
  --region ap-south-1 \
  --query "SecurityGroups[*].[GroupName,GroupId]" \
  --output table

export ALB_SG=sg-04fe39f58ede5c7d2
export EC2_SG=sg-076b6436993e856b5
echo "ALB_SG=$ALB_SG  EC2_SG=$EC2_SG"
ALB_SG=sg-04fe39f58ede5c7d2  EC2_SG=sg-076b6436993e856b5


  echo "ALB_SG=$ALB_SG  EC2_SG=$EC2_SG
  
```

---

## Checkpoint

- [ ] `alb-sg` allows inbound TCP 80 from `0.0.0.0/0`
- [ ] `ec2-sg` allows inbound TCP 5000 from `alb-sg` (a group, not a CIDR)
- [ ] Neither group opens port 22
- [ ] Both groups belong to `webapp-vpc`

---

**Next:** [Step 3 — IAM Roles](./03-iam-roles.md)
