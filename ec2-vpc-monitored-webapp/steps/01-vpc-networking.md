# Step 1 — VPC: Build the Network Foundation

Everything else sits inside this network. You'll build a VPC with **two public subnets**
(for the ALB and NAT Gateway) and **two private subnets** (for the EC2 instances), spread
across two Availability Zones so the load balancer can be highly available.

---

## 1.1 What You're Creating

```
VPC:                webapp-vpc  (10.0.0.0/16)
├── Public Subnet A:   webapp-public-a   (10.0.1.0/24)  — us-east-1a
├── Public Subnet B:   webapp-public-b   (10.0.2.0/24)  — us-east-1b
├── Private Subnet A:  webapp-private-a  (10.0.11.0/24) — us-east-1a
├── Private Subnet B:  webapp-private-b  (10.0.12.0/24) — us-east-1b
├── Internet Gateway:  webapp-igw
├── NAT Gateway:       webapp-nat        (in Public Subnet A, needs an Elastic IP)
├── Public Route Table:  webapp-rt-public   (0.0.0.0/0 → IGW)
└── Private Route Table: webapp-rt-private  (0.0.0.0/0 → NAT)
```

**Why public *and* private subnets?**
The ALB must be reachable from the internet, so it lives in **public** subnets. The EC2
instances should **not** be directly reachable, so they live in **private** subnets with
no public IP. They still need outbound internet (to install packages and pull code), which
they get through the **NAT Gateway** — outbound only, never inbound.

**Why two AZs?** An ALB requires subnets in at least two Availability Zones, and spreading
instances across AZs means one AZ failure doesn't take you down.

---

## 1.2 Console — Create the VPC

1. Open the **VPC** console → **Create VPC**.
2. Choose **VPC only**.

   | Field | Value |
   |-------|-------|
   | Name tag | `webapp-vpc` |
   | IPv4 CIDR | `10.0.0.0/16` |
   | Tenancy | Default |

3. Click **Create VPC**.

---

## 1.3 Console — Create the Four Subnets

**Subnets → Create subnet** → select `webapp-vpc`, then add all four:

| Subnet name | AZ | IPv4 CIDR |
|-------------|----|-----------|
| `webapp-public-a` | us-east-1a | `10.0.1.0/24` |
| `webapp-public-b` | us-east-1b | `10.0.2.0/24` |
| `webapp-private-a` | us-east-1a | `10.0.11.0/24` |
| `webapp-private-b` | us-east-1b | `10.0.12.0/24` |

For **each public subnet** only: select it → **Actions → Edit subnet settings** →
check **Enable auto-assign public IPv4 address** → **Save**. (Private subnets stay off.)

---

## 1.4 Console — Internet Gateway

1. **Internet gateways → Create internet gateway** → Name `webapp-igw` → **Create**.
2. **Actions → Attach to VPC** → `webapp-vpc`.

---

## 1.5 Console — NAT Gateway

1. **NAT gateways → Create NAT gateway**.

   | Field | Value |
   |-------|-------|
   | Name | `webapp-nat` |
   | Subnet | `webapp-public-a` (NAT lives in a **public** subnet) |
   | Connectivity | Public |
   | Elastic IP | Click **Allocate Elastic IP** |

2. Click **Create NAT gateway**. It takes a few minutes to become **Available**.

> The NAT Gateway is the one always-on cost in this project (~$0.045/hr). It exists so
> private instances can reach the internet outbound without being reachable inbound.

---

## 1.6 Console — Route Tables

**Public route table:**
1. **Route tables → Create route table** → Name `webapp-rt-public`, VPC `webapp-vpc`.
2. **Routes → Edit routes → Add route**: `0.0.0.0/0` → **Internet Gateway** `webapp-igw`.
3. **Subnet associations → Edit** → select both **public** subnets.

**Private route table:**
1. **Create route table** → Name `webapp-rt-private`, VPC `webapp-vpc`.
2. **Routes → Edit routes → Add route**: `0.0.0.0/0` → **NAT Gateway** `webapp-nat`.
3. **Subnet associations → Edit** → select both **private** subnets.

---

## 1.7 AWS CLI (Alternative — Full Setup)

```bash
REGION=us-east-1

VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
  --query 'Vpc.VpcId' --output text --region $REGION)
aws ec2 create-tags --resources $VPC_ID --tags Key=Name,Value=webapp-vpc --region $REGION


aws ec2 describe-vpcs \
  --region ap-south-1 \
  --query "Vpcs[*].[VpcId,Tags[?Key=='Name']|[0].Value]" \
  --output table


# Subnets
PUB_A=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a --query 'Subnet.SubnetId' --output text --region $REGION)
PUB_B=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.2.0/24 \
  --availability-zone us-east-1b --query 'Subnet.SubnetId' --output text --region $REGION)
PRIV_A=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.11.0/24 \
  --availability-zone us-east-1a --query 'Subnet.SubnetId' --output text --region $REGION)
PRIV_B=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.12.0/24 \
  --availability-zone us-east-1b --query 'Subnet.SubnetId' --output text --region $REGION)
aws ec2 modify-subnet-attribute --subnet-id $PUB_A --map-public-ip-on-launch --region $REGION
aws ec2 modify-subnet-attribute --subnet-id $PUB_B --map-public-ip-on-launch --region $REGION

# Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' \
  --output text --region $REGION)
aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID --region $REGION

# NAT Gateway (needs an Elastic IP)
EIP_ALLOC=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text --region $REGION)
NAT_ID=$(aws ec2 create-nat-gateway --subnet-id $PUB_A --allocation-id $EIP_ALLOC \
  --query 'NatGateway.NatGatewayId' --output text --region $REGION)
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_ID --region $REGION

# Route tables
RT_PUB=$(aws ec2 create-route-table --vpc-id $VPC_ID --query 'RouteTable.RouteTableId' --output text --region $REGION)
aws ec2 create-route --route-table-id $RT_PUB --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID --region $REGION
aws ec2 associate-route-table --route-table-id $RT_PUB --subnet-id $PUB_A --region $REGION
aws ec2 associate-route-table --route-table-id $RT_PUB --subnet-id $PUB_B --region $REGION

RT_PRIV=$(aws ec2 create-route-table --vpc-id $VPC_ID --query 'RouteTable.RouteTableId' --output text --region $REGION)
aws ec2 create-route --route-table-id $RT_PRIV --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_ID --region $REGION
aws ec2 associate-route-table --route-table-id $RT_PRIV --subnet-id $PRIV_A --region $REGION
aws ec2 associate-route-table --route-table-id $RT_PRIV --subnet-id $PRIV_B --region $REGION

echo "VPC=$VPC_ID  PUB_A=$PUB_A PUB_B=$PUB_B  PRIV_A=$PRIV_A PRIV_B=$PRIV_B"
```

> **Save these IDs.** You'll reuse them in Steps 4–6.

---

## Checkpoint

- [ ] VPC `webapp-vpc` (10.0.0.0/16) exists
- [ ] 2 public subnets (auto-assign public IP ON) + 2 private subnets (OFF)
- [ ] Internet Gateway attached
- [ ] NAT Gateway is **Available** with an Elastic IP
- [ ] Public route table: `0.0.0.0/0 → IGW`, associated with both public subnets
- [ ] Private route table: `0.0.0.0/0 → NAT`, associated with both private subnets

---

**Next:** [Step 2 — Security Groups](./02-security-groups.md)
