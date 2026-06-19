# Step 8 — SNS: Turn Alarms Into Emails

An alarm that no human sees is useless. **SNS** (Simple Notification Service) is the
pub/sub service that delivers the alarm to your inbox. You'll create a topic, subscribe
your email, and wire the alarm's actions to it.

---

## 8.1 Console — Create the Topic and Subscribe

1. **SNS console → Topics → Create topic**.

   | Field | Value |
   |-------|-------|
   | Type | **Standard** |
   | Name | `webapp-alerts` |

2. **Create topic.**
3. On the topic page → **Create subscription**:

   | Field | Value |
   |-------|-------|
   | Protocol | **Email** |
   | Endpoint | `you@example.com` |

4. **Create subscription.** AWS sends a **confirmation email** — click the link in it.
   Until you confirm, the subscription is **Pending** and delivers nothing.

---

## 8.2 Console — Wire the Alarm to SNS

1. **CloudWatch → Alarms → `webapp-high-cpu` → Actions → Edit**.
2. Step through to **Notification**: **In alarm** → send to SNS topic `webapp-alerts`.
3. (Optional but nice) Add an **OK** action to the same topic, so you also get an email when
   CPU recovers. **Update alarm.**

---

## 8.3 Why SNS (and not "just email from the alarm")?

CloudWatch alarms can't email directly — they publish to SNS, and SNS fans the message out
to *every* subscriber and protocol (email, SMS, Lambda, an HTTP webhook, an SQS queue…).
That indirection means you can later add a Slack webhook or a PagerDuty integration to the
same topic without touching the alarm. It's the same fanout pattern as the
[sqs-sns-iam-messaging](../../sqs-sns-iam-messaging/README.md) project.

---

## 8.4 End-to-End Test

```bash
ALB=http://<ALB-DNS-name>
for i in $(seq 1 20); do curl -s "$ALB/api/load?seconds=8" >/dev/null & done; wait
```

Within a few minutes you should receive an **email** titled
*"ALARM: webapp-high-cpu in US East (N. Virginia)"*. When load drops and the OK action
fires, a recovery email follows.

---

## 8.5 AWS CLI (Alternative)

```bash
REGION=us-east-1
TOPIC_ARN=$(aws sns create-topic --name webapp-alerts --query 'TopicArn' --output text --region $REGION)

harish@Harish:~/scripts$ echo $TOPIC_ARN
arn:aws:sns:ap-south-1:495013583028:webapp-alerts
harish@Harish:~/scripts$


aws sns subscribe --topic-arn $TOPIC_ARN --protocol email --notification-endpoint you@example.com --region $REGION
# ... confirm the email link, then attach to the alarm:
aws cloudwatch put-metric-alarm --alarm-name webapp-high-cpu \
  --alarm-description "ASG avg CPU > 60% for 2 min" \
  --namespace AWS/EC2 --metric-name CPUUtilization \
  --dimensions Name=AutoScalingGroupName,Value=webapp-asg \
  --statistic Average --period 60 --evaluation-periods 2 \
  --threshold 60 --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions $TOPIC_ARN --ok-actions $TOPIC_ARN --region $REGION


  harish@Harish:~/scripts$ aws sns subscribe --topic-arn $TOPIC_ARN --protocol email --notification-endpoint you@example.com --region $REGION
# ... confirm the email link, then attach to the alarm:
aws cloudwatch put-metric-alarm --alarm-name webapp-high-cpu \
  --alarm-description "ASG avg CPU > 60% for 2 min" \
  --namespace AWS/EC2 --metric-name CPUUtilization \
  --dimensions Name=AutoScalingGroupName,Value=webapp-asg \
  --statistic Average --period 60 --evaluation-periods 2 \
  --threshold 60 --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions $TOPIC_ARN --ok-actions $TOPIC_ARN --region $REGION
{
    "SubscriptionArn": "pending confirmation"
}
harish@Harish:~/scripts$

```

harish@Harish:~/scripts$ aws sns list-subscriptions-by-topic \
  --topic-arn $TOPIC_ARN \
  --region $REGION
{
    "Subscriptions": [
        {
            "SubscriptionArn": "PendingConfirmation",
            "Owner": "495013583028",
            "Protocol": "email",
            "Endpoint": "you@example.com",
            "TopicArn": "arn:aws:sns:ap-south-1:495013583028:webapp-alerts"
        }
    ]
}
harish@Harish:~/scripts$


Subscribe with the New Email:

aws sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol email \
  --notification-endpoint senhrish108@gmail.com \
  --region $REGION

harish@Harish:~/scripts$ aws sns list-subscriptions-by-topic   --topic-arn $TOPIC_ARN   --region $REGION
{
    "Subscriptions": [
        {
            "SubscriptionArn": "PendingConfirmation",
            "Owner": "495013583028",
            "Protocol": "email",
            "Endpoint": "you@example.com",
            "TopicArn": "arn:aws:sns:ap-south-1:495013583028:webapp-alerts"
        },
        {
            "SubscriptionArn": "PendingConfirmation",
            "Owner": "495013583028",
            "Protocol": "email",
            "Endpoint": "senhrish108@gmail.com",
            "TopicArn": "arn:aws:sns:ap-south-1:495013583028:webapp-alerts"
        }
    ]
}
harish@Harish:~/scripts$

> `setup_monitoring.py` from Step 7 already does all of this — this is the manual equivalent.

---

## Checkpoint

- [ ] SNS topic `webapp-alerts` exists
- [ ] Your email subscription is **Confirmed** (not Pending)
- [ ] `webapp-high-cpu` alarm action points to `webapp-alerts`
- [ ] Driving `/api/load` produced an alarm email (and an OK email on recovery)

---

**Next:** [Step 9 — CloudTrail Audit](./09-cloudtrail-audit.md)
