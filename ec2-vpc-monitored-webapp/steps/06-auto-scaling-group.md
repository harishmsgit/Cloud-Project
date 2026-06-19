# Step 6 — Auto Scaling Group

The ASG turns "a launch template" into "a self-healing, elastic fleet". It keeps a desired
number of instances running across both AZs, registers them into the target group, replaces
unhealthy ones, and scales out when CPU climbs.

---

## 6.1 Console — Create the ASG

1. **EC2 console → Auto Scaling Groups → Create Auto Scaling group**.
2. **Name:** `webapp-asg`. **Launch template:** `webapp-lt` (Latest version). Next.
3. **Network:**
   - VPC: `webapp-vpc`
   - Subnets: both **private** subnets (`webapp-private-a`, `webapp-private-b`)

   > Instances go in **private** subnets — no public IP. The ALB (public) reaches them.

4. **Load balancing:** Attach to an existing load balancer → **Choose from target groups**
   → `webapp-tg`.
5. **Health checks:** Turn on **ELB** health checks (in addition to EC2). Grace period
   `120` seconds — gives user-data time to finish before the ALB starts judging health.
6. **Group size:**

   | Setting | Value |
   |---------|-------|
   | Desired capacity | 2 |
   | Minimum capacity | 1 |
   | Maximum capacity | 4 |

7. **Scaling policy → Target tracking:** Metric **Average CPU utilization**, Target value
   **50**. (The ASG adds instances to keep average CPU near 50%.)
8. Next through the rest → **Create Auto Scaling group**.

---

## 6.2 Verify Traffic Flows End to End

1. Wait ~3 minutes. In **Target Groups → `webapp-tg` → Targets**, both instances should
   become **healthy**.
2. Open `http://<ALB-DNS-name>/` in a browser — you should see the JSON home response.
3. Refresh `http://<ALB-DNS-name>/api/info` a few times; the `instance_id` should alternate
   between the two instances — proof the ALB is load-balancing.

---

## 6.3 Watch It Scale (preview of monitoring)

Generate load against the CPU-burning endpoint and watch the ASG react:

```bash
ALB=http://<ALB-DNS-name>
# fire 20 parallel 8-second CPU burns
for i in $(seq 1 20); do curl -s "$ALB/api/load?seconds=8" >/dev/null & done; wait
```

Within a few minutes the **Activity** tab of the ASG shows a scale-out event. You'll wire
the *alarm + email* side of this in Steps 7–8.

---

## 6.4 AWS CLI (Alternative)

```bash
REGION=us-east-1
# PRIV_A, PRIV_B, TG_ARN from earlier steps
harish@Harish:~$ aws ec2 describe-security-groups \
  --region ap-south-1 \
  --query "SecurityGroups[*].[GroupId,GroupName,VpcId]" \
  --output table

harish@Harish:~$ aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=ec2-sg,alb-sg" \
  --region ap-south-1 \
  --query "SecurityGroups[*].[GroupId,GroupName,VpcId]" \
  --output table
-------------------------------------------------------------
|                  DescribeSecurityGroups                   |
+-----------------------+---------+-------------------------+
|  sg-076b6436993e856b5 |  ec2-sg |  vpc-06bbc38be0501663f  |
|  sg-04fe39f58ede5c7d2 |  alb-sg |  vpc-06bbc38be0501663f  |
|  sg-07908bea5300ce1fb |  alb-sg |  vpc-0edde88b9ffe9c2c6  |
+-----------------------+---------+-------------------------+
harish@Harish:~$


harish@Harish:~$
harish@Harish:~$ aws ec2 describe-security-groups \
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
harish@Harish:~$ aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=vpc-06bbc38be0501663f" \
  --region ap-south-1 \
  --query "Subnets[*].[SubnetId,AvailabilityZone,MapPublicIpOnLaunch]" \
  --output table
------------------------------------------------------
|                   DescribeSubnets                  |
+---------------------------+---------------+--------+
|  subnet-0e54df2d816960e00 |  ap-south-1a  |  False |
|  subnet-01a80965fe86324ed |  ap-south-1b  |  False |
|  subnet-0a41d7a8d85ac7a34 |  ap-south-1b  |  True  |
|  subnet-02fee81603fe57089 |  ap-south-1a  |  True  |
+---------------------------+---------------+--------+
harish@Harish:~$

export PRIV_A=subnet-0e54df2d816960e00
export PRIV_B=subnet-01a80965fe86324ed


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

export TG_ARN=arn:aws:elasticloadbalancing:ap-south-1:495013583028:targetgroup/webapp-tg-eks/5f9b3e7a25ef493b


aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name webapp-asg \
  --launch-template LaunchTemplateName=webapp-lt,Version='$Latest' \
  --vpc-zone-identifier "$PRIV_A,$PRIV_B" \
  --target-group-arns $TG_ARN \
  --health-check-type ELB --health-check-grace-period 120 \
  --min-size 1 --max-size 4 --desired-capacity 2 --region $REGION


  harish@Harish:~$ aws autoscaling describe-auto-scaling-groups \
  --region ap-south-1 \
  --query "AutoScalingGroups[*].[AutoScalingGroupName,VPCZoneIdentifier]" \
  --output table
----------------------------------------------------------------------------------------------------------------------
|                                              DescribeAutoScalingGroups                                             |
+--------------------------------------------------------------+-----------------------------------------------------+
|  eks-dev-webapp-workers-5ecf5b3d-816a-a13c-56f5-a4aea5c5485b |  subnet-0c5e50e1269a8aebb,subnet-0f51a3c732212810d  |
|  webapp-asg                                                  |  subnet-0e54df2d816960e00,subnet-01a80965fe86324ed  |
+--------------------------------------------------------------+-----------------------------------------------------+
harish@Harish:~$

aws autoscaling put-scaling-policy \
  --auto-scaling-group-name webapp-asg \
  --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {"PredefinedMetricType": "ASGAverageCPUUtilization"},
    "TargetValue": 50.0
  }' --region $REGION
```

harish@Harish:~$ aws autoscaling put-scaling-policy \
  --auto-scaling-group-name webapp-asg \
  --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {"PredefinedMetricType": "ASGAverageCPUUtilization"},
    "TargetValue": 50.0
  }' --region $REGION
{
    "PolicyARN": "arn:aws:autoscaling:ap-south-1:495013583028:scalingPolicy:1a7fb2a9-4bd0-440c-830d-cfabd04d6462:autoScalingGroupName/webapp-asg:policyName/cpu-target-tracking",
    "Alarms": [
        {
            "AlarmName": "TargetTracking-webapp-asg-AlarmHigh-9d6d755b-deb9-4826-9530-b972fbe55890",
            "AlarmARN": "arn:aws:cloudwatch:ap-south-1:495013583028:alarm:TargetTracking-webapp-asg-AlarmHigh-9d6d755b-deb9-4826-9530-b972fbe55890"
        },
        {
            "AlarmName": "TargetTracking-webapp-asg-AlarmLow-559c9f03-c05e-4e42-9450-ef2ba333b6a3",
            "AlarmARN": "arn:aws:cloudwatch:ap-south-1:495013583028:alarm:TargetTracking-webapp-asg-AlarmLow-559c9f03-c05e-4e42-9450-ef2ba333b6a3"
        }
    ]
}
harish@Harish:~$

---

## Checkpoint

- [ ] ASG `webapp-asg` runs in both **private** subnets
- [ ] Desired 2 / Min 1 / Max 4
- [ ] Attached to `webapp-tg`; ELB health checks on; grace period 120 s
- [ ] Both targets show **healthy**
- [ ] ALB DNS serves the app; `instance_id` alternates across refreshes
- [ ] Target-tracking policy on Average CPU @ 50% exists

---

**Next:** [Step 7 — CloudWatch Monitoring](./07-cloudwatch-monitoring.md)
