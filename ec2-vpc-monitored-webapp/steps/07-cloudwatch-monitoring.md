# Step 7 — CloudWatch: Metrics, Alarm, Dashboard

EC2 publishes CPU, network, and disk-I/O metrics for free — but **not memory**, because
that lives inside the OS. The **CloudWatch agent** (installed by user-data) fills that gap.
Here you'll configure the agent, create a high-CPU **alarm**, and build a **dashboard**.

---

## 7.1 Push the CloudWatch Agent Config

The agent config is in `scripts/cloudwatch-agent-config.json`. It collects
`mem_used_percent` and disk usage into a custom namespace `EC2MonitoredWebApp`, and ships
`/var/log/user-data.log` to a log group.

Store the config in **SSM Parameter Store**, then have the agent fetch it on every instance:

harish@Harish:~$ cd scripts/
harish@Harish:~/scripts$ nano ~/scripts/cloudwatch-agent-config.json

aws ssm put-parameter \
  --name "/webapp/cloudwatch-agent-config" \
  --type String \
  --value file://$(pwd)/cloudwatch-agent-config.json \    ##--value file:///home/harish/scripts/cloudwatch-agent-config.json \
  --overwrite \
  --region ap-south-1

  harish@Harish:~/scripts$ aws ssm put-parameter \
  --name "/webapp/cloudwatch-agent-config" \
  --type String \
  --value file://$(pwd)/cloudwatch-agent-config.json \
  --overwrite \
  --region ap-south-1
{
    "Version": 1,
    "Tier": "Standard"
}
harish@Harish:~/scripts$ ls
cloudwatch-agent-config.json
harish@Harish:~/scripts$


```bash
REGION=us-east-1

aws ssm put-parameter --name "/webapp/cloudwatch-agent-config" --type String \
  --value file://scripts/cloudwatch-agent-config.json --overwrite --region $REGION

# Tell the agent on every running instance to load it (no SSH — SSM Run Command):
aws ssm send-command \
  --document-name "AmazonCloudWatch-ManageAgent" \
  --targets "Key=tag:Project,Values=webapp" \
  --parameters '{"action":["configure"],"mode":["ec2"],
    "optionalConfigurationSource":["ssm"],
    "optionalConfigurationLocation":["/webapp/cloudwatch-agent-config"],
    "optionalRestart":["yes"]}' \
  --region $REGION
```

After a minute, **CloudWatch → Metrics → All metrics → EC2MonitoredWebApp** shows
`MemoryUtilization` per instance.

---

## 7.2 Console — Create the High-CPU Alarm

1. **CloudWatch → Alarms → Create alarm → Select metric**.
2. **EC2 → By Auto Scaling Group → `webapp-asg` → CPUUtilization** → Select metric.
3. Conditions:

   | Field | Value |
   |-------|-------|
   | Statistic | Average |
   | Period | 1 minute |
   | Threshold type | Static |
   | Whenever CPUUtilization is | **Greater than 60** |
   | Datapoints to alarm | **2 out of 2** |

4. **Next → Notification:** you'll attach the SNS topic in Step 8. For now choose **Remove**
   the notification (or come back after Step 8). **Alarm name:** `webapp-high-cpu`.
5. **Create alarm.**

aws cloudwatch put-metric-alarm \
  --alarm-name HighCPUAlarm \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 60 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:ap-south-1:123456789012:NotifyMe \
  --region ap-south-1

harish@Harish:~/scripts$ aws cloudwatch describe-alarms   --alarm-names HighCPUAlarm   --region ap-south-1
{
    "MetricAlarms": [
        {
            "AlarmName": "HighCPUAlarm",
            "AlarmArn": "arn:aws:cloudwatch:ap-south-1:495013583028:alarm:HighCPUAlarm",
            "AlarmConfigurationUpdatedTimestamp": "2026-06-18T19:20:41.227000+00:00",
            "ActionsEnabled": true,
            "OKActions": [],
            "AlarmActions": [
                "arn:aws:sns:ap-south-1:123456789012:NotifyMe"
            ],
            "InsufficientDataActions": [],
            "StateValue": "INSUFFICIENT_DATA",
            "StateReason": "Unchecked: Initial alarm creation",
            "StateUpdatedTimestamp": "2026-06-18T19:20:41.227000+00:00",
            "MetricName": "CPUUtilization",
            "Namespace": "AWS/EC2",
            "Statistic": "Average",
            "Dimensions": [
                {
                    "Name": "InstanceId",
                    "Value": "i-1234567890abcdef0"
                }
            ],
            "Period": 60,
            "EvaluationPeriods": 2,
            "Threshold": 80.0,
            "ComparisonOperator": "GreaterThanThreshold",
            "StateTransitionedTimestamp": "2026-06-18T19:20:41.227000+00:00"
        }
    ],
    "CompositeAlarms": []
}
harish@Harish:~/scripts$

---

## 7.3 Console — Build the Dashboard

1. **CloudWatch → Dashboards → Create dashboard** → name `webapp-monitoring`.
2. Add three widgets (**Add widget → Line**):
   - **ASG Average CPU** — `AWS/EC2 → CPUUtilization → AutoScalingGroupName webapp-asg`
   - **Memory** — `EC2MonitoredWebApp → MemoryUtilization`
   - **ALB Request Count** — `AWS/ApplicationELB → RequestCount → LoadBalancer <alb-suffix>`
3. **Save dashboard.**

harish@Harish:~/scripts$ aws autoscaling put-scaling-policy \
  --policy-name scale-in-policy \
  --auto-scaling-group-name webapp-asg \
  --scaling-adjustment -1 \
  --adjustment-type ChangeInCapacity \
  --region ap-south-1
{
    "PolicyARN": "arn:aws:autoscaling:ap-south-1:495013583028:scalingPolicy:7835686f-4fdf-47d5-80cf-bec80b9756d5:autoScalingGroupName/webapp-asg:policyName/scale-in-policy",
    "Alarms": []
}
harish@Harish:~/scripts$

harish@Harish:~/scripts$ export SCALE_OUT_ARN=arn:aws:autoscaling:ap-south-1:495013583028:scalingPolicy:7835686f-4fdf-47d5-80cf-bec80
b9756d5:autoScalingGroupName/webapp-asg:policyName/scale-in-policy
harish@Harish:~/scripts$

Create a Scale‑In Policy:

harish@Harish:~/scripts$ aws autoscaling put-scaling-policy \
  --policy-name scale-in-policy \
  --auto-scaling-group-name webapp-asg \
  --scaling-adjustment -1 \
  --adjustment-type ChangeInCapacity \
  --region ap-south-1
{
    "PolicyARN": "arn:aws:autoscaling:ap-south-1:495013583028:scalingPolicy:7835686f-4fdf-47d5-80cf-bec80b9756d5:autoScalingGroupName/webapp-asg:policyName/scale-in-policy",
    "Alarms": []
}
harish@Harish:~/scripts$ export SCALE_IN_ARN=arn:aws:autoscaling:ap-south-1:495013583028:scalingPolicy:7835686f-4fdf-47d5-80cf-bec80b9756d5:autoScalingGroupName/webapp-asg:policyName/scale-in-policy
harish@Harish:~/scripts$

Attach Policies to Alarms:

aws cloudwatch put-metric-alarm \
  --alarm-name HighCPUAlarm \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 60 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=AutoScalingGroupName,Value=webapp-asg \
  --evaluation-periods 2 \
  --alarm-actions $SCALE_OUT_ARN \
  --region ap-south-1


Low CPU Alarm (scale in):

aws cloudwatch put-metric-alarm \
  --alarm-name LowCPUAlarm \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 60 \
  --threshold 30 \
  --comparison-operator LessThanThreshold \
  --dimensions Name=AutoScalingGroupName,Value=webapp-asg \
  --evaluation-periods 2 \
  --alarm-actions $SCALE_IN_ARN \
  --region ap-south-1

aws autoscaling describe-policies --auto-scaling-group-name webapp-asg --region ap-south-1
aws cloudwatch describe-alarms --region ap-south-1



---

## 7.4 Automation — Do 7.2–7.3 in One Command (Boto3)

The Console is good for learning; in real life you script it. `scripts/setup_monitoring.py`
creates the SNS topic (Step 8), the CPU alarm, **and** the dashboard in one idempotent run:

```bash
pip install boto3
python scripts/setup_monitoring.py \
  --asg-name webapp-asg \
  --alb-arn-suffix app/webapp-alb/0123456789abcdef \
  --email you@example.com
```

Re-running it is safe — every API call is create-or-update. This single script is the
"automation" deliverable: the entire monitoring stack as repeatable code.

---

## 7.5 Trip the Alarm

```bash
ALB=http://<ALB-DNS-name>
for i in $(seq 1 20); do curl -s "$ALB/api/load?seconds=8" >/dev/null & done; wait
```

Within ~2–3 minutes `webapp-high-cpu` flips to **In alarm** (red). Leave it — Step 8 turns
that red state into an email in your inbox.

---

## Checkpoint

- [ ] CloudWatch agent config delivered via SSM; `MemoryUtilization` visible in `EC2MonitoredWebApp`
- [ ] Alarm `webapp-high-cpu` exists (Avg CPU > 60% for 2 minutes)
- [ ] Dashboard `webapp-monitoring` shows CPU, memory, and ALB request count
- [ ] (Optional) `setup_monitoring.py` ran cleanly
- [ ] Driving `/api/load` flips the alarm to **In alarm**

---

**Next:** [Step 8 — SNS Alerts](./08-sns-alerts.md)
