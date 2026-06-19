# Step 5 — Application Load Balancer

The ALB is the single public entry point. It listens on port 80, health-checks each
instance on `/health`, and forwards healthy traffic to the app on port 5000. We create the
**target group** first (the ASG will register instances into it in Step 6), then the ALB
and its listener.

---

## 5.1 Console — Create the Target Group

1. **EC2 console → Target Groups → Create target group**.

   | Field | Value |
   |-------|-------|
   | Target type | **Instances** |
   | Name | `webapp-tg` |
   | Protocol / Port | HTTP / **5000** |
   | VPC | `webapp-vpc` |
   | Health check path | `/health` |

2. Under **Advanced health check settings**:

   | Field | Value | Why |
   |-------|-------|-----|
   | Healthy threshold | 2 | 2 good checks → mark healthy quickly |
   | Unhealthy threshold | 2 | 2 bad checks → pull it out fast |
   | Interval | 15 s | How often the ALB polls `/health` |
   | Success codes | 200 | Our `/health` returns 200 |

3. **Create target group**. Don't register any targets — the ASG does that in Step 6.

---

## 5.2 Console — Create the Load Balancer

1. **Load Balancers → Create load balancer → Application Load Balancer**.

   | Field | Value |
   |-------|-------|
   | Name | `webapp-alb` |
   | Scheme | **Internet-facing** |
   | IP address type | IPv4 |
   | VPC | `webapp-vpc` |
   | Mappings | Both **public** subnets (`webapp-public-a`, `webapp-public-b`) |
   | Security group | `alb-sg` (remove the default) |

2. **Listeners and routing:** Protocol **HTTP**, Port **80**, **Forward to** `webapp-tg`.
3. **Create load balancer**. Wait until **State = Active** (1–2 minutes).
4. Copy the **DNS name** (e.g. `webapp-alb-123456789.us-east-1.elb.amazonaws.com`).

---

## 5.3 AWS CLI (Alternative)

```bash
REGION=us-east-1
# VPC_ID, PUB_A, PUB_B, ALB_SG from earlier steps

TG_ARN=$(aws elbv2 create-target-group --name webapp-tg \
  --protocol HTTP --port 5000 --vpc-id $VPC_ID --target-type instance \
  --health-check-path /health --healthy-threshold-count 2 \
  --unhealthy-threshold-count 2 --health-check-interval-seconds 15 \
  --query 'TargetGroups[0].TargetGroupArn' --output text --region $REGION)

-----
sample data
echo $TG_ARN
arn:aws:elasticloadbalancing:ap-south-1:495013583028:targetgroup/webapp-tg/d3485cd7d0be433a

aws ec2 describe-subnets \
  --region ap-south-1 \
  --query "Subnets[*].[SubnetId,Tags[?Key=='Name']|[0].Value,MapPublicIpOnLaunch]" \
  --output table

aws ec2 describe-subnets \
  --subnet-ids subnet-0c5e50e1269a8aebb subnet-0f51a3c732212810d \
  --query "Subnets[*].[SubnetId,VpcId]" \
  --region ap-south-1 \
  --output table
-------------------------------------------------------
|                   DescribeSubnets                   |
+---------------------------+-------------------------+
|  subnet-0c5e50e1269a8aebb |  vpc-0edde88b9ffe9c2c6  |
|  subnet-0f51a3c732212810d |  vpc-0edde88b9ffe9c2c6  |
+---------------------------+-------------------------+

aws ec2 describe-security-groups \
  --group-ids sg-04fe39f58ede5c7d2 \
  --query "SecurityGroups[0].[GroupId,VpcId]" \
  --region ap-south-1 \
  --output table
---------------------------
| DescribeSecurityGroups  |
+-------------------------+
|  sg-04fe39f58ede5c7d2   |
|  vpc-06bbc38be0501663f  |
+-------------------------+

export PUB_A=subnet-0c5e50e1269a8aebb
export PUB_B=subnet-0f51a3c732212810d

aws ec2 describe-security-groups \
  --filters "Name=vpc-id,Values=vpc-0edde88b9ffe9c2c6" \
  --query "SecurityGroups[*].[GroupName,GroupId]" \
  --region ap-south-1 \
  --output table

----------

ALB_ARN=$(aws elbv2 create-load-balancer --name webapp-alb \
  --subnets $PUB_A $PUB_B --security-groups $ALB_SG --scheme internet-facing \
  --type application --query 'LoadBalancers[0].LoadBalancerArn' --output text --region $REGION)

aws elbv2 create-listener --load-balancer-arn $ALB_ARN \
  --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN --region $REGION

aws elbv2 describe-load-balancers --load-balancer-arns $ALB_ARN \
  --query 'LoadBalancers[0].DNSName' --output text --region $REGION
echo "TG_ARN=$TG_ARN  ALB_ARN=$ALB_ARN"
```

> **Save `TG_ARN`** — the ASG needs it in Step 6. Also note the **ALB ARN suffix** (the
> part after `loadbalancer/`, e.g. `app/webapp-alb/0123...`) for the dashboard in Step 7.

---

## Checkpoint

- [ ] Target group `webapp-tg` (HTTP 5000, health check `/health`, 200) exists
- [ ] ALB `webapp-alb` is **Active**, internet-facing, in both public subnets, using `alb-sg`
- [ ] Listener on port 80 forwards to `webapp-tg`
- [ ] You saved the ALB DNS name, `TG_ARN`, and the ALB ARN suffix
- [ ] Targets are still empty (expected until Step 6)

---


----
see the example:

Get the Security Group, VPC, and ALB details for your webapp-eks-cluster

******************************************************************************

harish@Harish:~$ aws eks describe-cluster \
  --name webapp-eks-cluster \
  --region ap-south-1 \
  --query "cluster.resourcesVpcConfig.vpcId" \
  --output text
vpc-0edde88b9ffe9c2c6

harish@Harish:~$ aws eks describe-cluster \
  --name webapp-eks-cluster \
  --region ap-south-1 \
  --query "cluster.resourcesVpcConfig.securityGroupIds" \
  --output text
sg-07fb090636a2a3414
harish@Harish:~$


harish@Harish:~$ aws eks describe-nodegroup \
  --cluster-name webapp-eks-cluster \
  --nodegroup-name dev-webapp-workers \
  --region ap-south-1 \
  --query "nodegroup.resources.securityGroups" \
  --output text
None
harish@Harish:~$


aws elbv2 describe-load-balancers \
  --region ap-south-1 \
  --query "LoadBalancers[?VpcId=='vpc-0edde88b9ffe9c2c6'].[LoadBalancerName,LoadBalancerArn,DNSName,VpcId]" \
  --output table



harish@Harish:~$ aws ec2 describe-security-groups \
  --filters "Name=vpc-id,Values=vpc-0edde88b9ffe9c2c6" \
  --query "SecurityGroups[*].[GroupName,GroupId]" \
  --region ap-south-1 \
  --output table
-------------------------------------------------------------------------
|                        DescribeSecurityGroups                         |
+----------------------------------------------+------------------------+
|  dev-webapp-eks-sg                           |  sg-07fb090636a2a3414  |
|  eks-cluster-sg-webapp-eks-cluster-127837498 |  sg-0a676162f8013ad92  |
|  dev-webapp-ec2-sg                           |  sg-053c31b8be2325397  |
|  default                                     |  sg-0670c407a6428e49d  |
+----------------------------------------------+------------------------+
harish@Harish:~$
harish@Harish:~$
harish@Harish:~$
harish@Harish:~$ aws ec2 create-security-group \
  --group-name alb-sg \
  --description "ALB ingress from internet" \
  --vpc-id vpc-0edde88b9ffe9c2c6 \
  --region ap-south-1
{
    "GroupId": "sg-07908bea5300ce1fb",
    "SecurityGroupArn": "arn:aws:ec2:ap-south-1:495013583028:security-group/sg-07908bea5300ce1fb"
}
harish@Harish:~$'


harish@Harish:~$ aws elbv2 describe-target-groups \
  --names webapp-tg \
  --region ap-south-1 \
  --query "TargetGroups[*].[TargetGroupName,TargetGroupArn,Port,Protocol,TargetType,VpcId]" \
  --output table
-------------------------------------------------------------------------------------------------------------------------------------------------------------------
|                                                                      DescribeTargetGroups                                                                       |
+-----------+-----------------------------------------------------------------------------------------------+-------+-------+-----------+-------------------------+
|  webapp-tg|  arn:aws:elasticloadbalancing:ap-south-1:495013583028:targetgroup/webapp-tg/d3485cd7d0be433a  |  5000 |  HTTP |  instance |  vpc-06bbc38be0501663f  |
+-----------+-----------------------------------------------------------------------------------------------+-------+-------+-----------+-------------------------+
harish@Harish:~$




harish@Harish:~$ TG_ARN=$(aws elbv2 create-target-group \
  --name webapp-tg-eks \
  --protocol HTTP \
  --port 80 \
  --vpc-id vpc-0edde88b9ffe9c2c6 \
  --target-type instance \
  --query 'TargetGroups[0].TargetGroupArn' \
  --output text \
  --region ap-south-1)
harish@Harish:~$
harish@Harish:~$
harish@Harish:~$
harish@Harish:~$ echo $TG_ARN
arn:aws:elasticloadbalancing:ap-south-1:495013583028:targetgroup/webapp-tg-eks/5f9b3e7a25ef493b
harish@Harish:~$

harish@Harish:~$ aws ec2 describe-instances \
  --filters "Name=vpc-id,Values=vpc-0edde88b9ffe9c2c6" \
  --query "Reservations[*].Instances[*].[InstanceId,PrivateIpAddress,State.Name]" \
  --region ap-south-1 \
  --output table
--------------------------------------------------
|                DescribeInstances               |
+----------------------+--------------+----------+
|  i-04b1a45430bbb7185 |  10.0.1.247  |  running |
|  i-04c9c1b6d39c3d823 |  10.0.1.122  |  running |
|  i-01063a03ddb6182b4 |  10.0.2.75   |  running |
+----------------------+--------------+----------+
harish@Harish:~$

harish@Harish:~$ aws elbv2 describe-target-groups \
  --region ap-south-1 \
  --query "TargetGroups[*].[TargetGroupName,TargetGroupArn,VpcId]" \
  --output table
-----------------------------------------------------------------------------------------------------------------------------------------------
|                                                            DescribeTargetGroups                                                             |
+---------------+---------------------------------------------------------------------------------------------------+-------------------------+
|  webapp-tg    |  arn:aws:elasticloadbalancing:ap-south-1:495013583028:targetgroup/webapp-tg/d3485cd7d0be433a      |  vpc-06bbc38be0501663f  |
|  webapp-tg-eks|  arn:aws:elasticloadbalancing:ap-south-1:495013583028:targetgroup/webapp-tg-eks/5f9b3e7a25ef493b  |  vpc-0edde88b9ffe9c2c6  |
+---------------+---------------------------------------------------------------------------------------------------+-------------------------+
harish@Harish:~$



Register All Three EC2 Instances:

harish@Harish:~$ aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:ap-south-1:495013583028:targetgroup/webapp-tg-eks/5f9b3e7a25ef493b \
  --targets Id=i-04b1a45430bbb7185 Id=i-04c9c1b6d39c3d823 Id=i-01063a03ddb6182b4 \
  --region ap-south-1
harish@Harish:~$
harish@Harish:~$


Create Listener:

harish@Harish:~$ aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:ap-south-1:495013583028:loadbalancer/app/webapp-alb/2cef235c4a288e73 \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:ap-south-1:495013583028:targetgroup/webapp-tg-eks/5f9b3e7a25ef493b \
  --region ap-south-1
{
    "Listeners": [
        {
            "ListenerArn": "arn:aws:elasticloadbalancing:ap-south-1:495013583028:listener/app/webapp-alb/2cef235c4a288e73/0683959d26b0c6da",
            "LoadBalancerArn": "arn:aws:elasticloadbalancing:ap-south-1:495013583028:loadbalancer/app/webapp-alb/2cef235c4a288e73",
            "Port": 80,
            "Protocol": "HTTP",
            "DefaultActions": [
                {
                    "Type": "forward",
                    "TargetGroupArn": "arn:aws:elasticloadbalancing:ap-south-1:495013583028:targetgroup/webapp-tg-eks/5f9b3e7a25ef493b",
                    "ForwardConfig": {
                        "TargetGroups": [
                            {
                                "TargetGroupArn": "arn:aws:elasticloadbalancing:ap-south-1:495013583028:targetgroup/webapp-tg-eks/5f9b3e7a25ef493b",
                                "Weight": 1
                            }
                        ],
                        "TargetGroupStickinessConfig": {
                            "Enabled": false
                        }
                    }
                }
            ]
        }
    ]
}
harish@Harish:~$




Verify Target Health:


arish@Harish:~$ aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:ap-south-1:495013583028:targetgroup/webapp-tg-eks/5f9b3e7a25ef493b \
  --region ap-south-1 \
  --output table
-------------------------------------------------------------------------------------------------------
|                                        DescribeTargetHealth                                         |
+-----------------------------------------------------------------------------------------------------+
||                                     TargetHealthDescriptions                                      ||
|+---------------------------------------------------------------------------------------------------+|
||                                          HealthCheckPort                                          ||
|+---------------------------------------------------------------------------------------------------+|
||  80                                                                                               ||
|+---------------------------------------------------------------------------------------------------+|
|||                                     AdministrativeOverride                                      |||
||+--------------------------------------------+-------------------------------------+--------------+||
|||                 Description                |               Reason                |    State     |||
||+--------------------------------------------+-------------------------------------+--------------+||
|||  No override is currently active on target |  AdministrativeOverride.NoOverride  |  no_override |||
||+--------------------------------------------+-------------------------------------+--------------+||
|||                                             Target                                              |||
||+-----------------------------------------------------------------------+-------------------------+||
|||                                  Id                                   |          Port           |||
||+-----------------------------------------------------------------------+-------------------------+||
|||  i-01063a03ddb6182b4                                                  |  80                     |||
||+-----------------------------------------------------------------------+-------------------------+||
|||                                          TargetHealth                                           |||
||+---------------------------------------------+-------------------------------------+-------------+||
|||                 Description                 |               Reason                |    State    |||
||+---------------------------------------------+-------------------------------------+-------------+||
|||  Target registration is in progress         |  Elb.RegistrationInProgress         |  initial    |||
||+---------------------------------------------+-------------------------------------+-------------+||
||                                     TargetHealthDescriptions                                      ||
|+---------------------------------------------------------------------------------------------------+|
||                                          HealthCheckPort                                          ||
|+---------------------------------------------------------------------------------------------------+|
||  80                                                                                               ||
|+---------------------------------------------------------------------------------------------------+|
|||                                     AdministrativeOverride                                      |||
||+--------------------------------------------+-------------------------------------+--------------+||
|||                 Description                |               Reason                |    State     |||
:





Get ALB DNS:

harish@Harish:~$ aws elbv2 describe-load-balancers \
  --load-balancer-arns arn:aws:elasticloadbalancing:ap-south-1:495013583028:loadbalancer/app/webapp-alb/2cef235c4a288e73 \
  --query "LoadBalancers[0].DNSName" \
  --region ap-south-1 \
  --output text
webapp-alb-156753393.ap-south-1.elb.amazonaws.com
harish@Harish:~$

----

**Next:** [Step 6 — Auto Scaling Group](./06-auto-scaling-group.md)


