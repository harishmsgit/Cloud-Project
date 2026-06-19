# Complete Configuration Guide: EC2-VPC-Monitored-WebApp

End-to-end setup for VPC, Security Groups, IAM Roles, EC2, ALB, ASG, CloudWatch, SNS, CloudTrail, and GitHub Actions deployment.

---

## Prerequisites

- **AWS Account** with appropriate permissions (Admin or custom policy)
- **Python 3.8+** installed and on PATH
- **AWS CLI v2+** configured with credentials (access key/secret or SSO)
- **Git** for version control
- **GitHub account** (for Actions CI/CD)
- **Optional:** Docker, jq, Terraform/CloudFormation experience

### Initial AWS CLI Setup

```powershell
# Configure a named profile
aws configure --profile myprofile
# Enter: Access Key ID, Secret Access Key, region (e.g., us-east-1), output (json)

# Verify credentials
aws sts get-caller-identity --profile myprofile
```

---

## Repository Structure

```
ec2-vpc-monitored-webapp/
├── CONFIGURATION.md              # This file
├── README.md
├── challenges.md
├── costs.md
├── troubleshooting.md
├── scripts/
│   ├── cloudwatch-agent-config.json    # CloudWatch agent configuration
│   ├── setup_monitoring.py             # Python script to setup monitoring
│   └── user-data.sh                    # EC2 user data bootstrap script
├── src/
│   ├── app.py                          # Flask/Python web application
│   ├── requirements.txt                # Python dependencies
│   └── test_app.py                     # Application tests
└── steps/                              # Reference documentation
    ├── 01-vpc-networking.md
    ├── 02-security-groups.md
    ├── 03-iam-roles.md
    ├── 04-launch-ec2.md
    ├── 05-application-load-balancer.md
    ├── 06-auto-scaling-group.md
    ├── 07-cloudwatch-monitoring.md
    ├── 08-sns-alerts.md
    ├── 09-cloudtrail-audit.md
    ├── 10-github-actions-deploy.md
    └── 11-cleanup.md
```

---

## Local Development Setup

### 1. Clone the Repository

```powershell
git clone https://github.com/your-org/ec2-vpc-monitored-webapp.git
cd ec2-vpc-monitored-webapp
```

### 2. Create Python Virtual Environment

**Windows (PowerShell):**

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install --upgrade pip
pip install -r src/requirements.txt
```

**macOS/Linux (bash):**

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -r src/requirements.txt
```

### 3. Test Application Locally

```powershell
cd src
python app.py
# Access http://localhost:5000 (or configured port)

# Run tests
pytest -v
```

---

## AWS Infrastructure Setup (Step-by-Step)

### Step 1: VPC Networking

Creates the isolated network environment for all resources.

#### Create VPC

```powershell
# Variables
$VPC_NAME = "webapp-vpc"
$VPC_CIDR = "10.0.0.0/16"
$REGION = "us-east-1"
$PROFILE = "myprofile"

# Create VPC
$VPC_ID = aws ec2 create-vpc `
  --cidr-block $VPC_CIDR `
  --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=$VPC_NAME}]" `
  --region $REGION `
  --profile $PROFILE `
  --query 'Vpc.VpcId' `
  --output text

Write-Host "VPC ID: $VPC_ID"

# Enable DNS hostnames
aws ec2 modify-vpc-attribute `
  --vpc-id $VPC_ID `
  --enable-dns-hostnames `
  --region $REGION `
  --profile $PROFILE
```

#### Create Subnets (Public & Private)

```powershell
# Public Subnet (for ALB)
$PUBLIC_SUBNET = aws ec2 create-subnet `
  --vpc-id $VPC_ID `
  --cidr-block "10.0.1.0/24" `
  --availability-zone "${REGION}a" `
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=public-subnet-1a}]" `
  --region $REGION `
  --profile $PROFILE `
  --query 'Subnet.SubnetId' `
  --output text

Write-Host "Public Subnet: $PUBLIC_SUBNET"

# Private Subnet (for EC2 instances)
$PRIVATE_SUBNET = aws ec2 create-subnet `
  --vpc-id $VPC_ID `
  --cidr-block "10.0.10.0/24" `
  --availability-zone "${REGION}b" `
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=private-subnet-1b}]" `
  --region $REGION `
  --profile $PROFILE `
  --query 'Subnet.SubnetId' `
  --output text

Write-Host "Private Subnet: $PRIVATE_SUBNET"

# Create additional public/private subnets for ALB redundancy
$PUBLIC_SUBNET_2 = aws ec2 create-subnet `
  --vpc-id $VPC_ID `
  --cidr-block "10.0.2.0/24" `
  --availability-zone "${REGION}b" `
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=public-subnet-1b}]" `
  --region $REGION `
  --profile $PROFILE `
  --query 'Subnet.SubnetId' `
  --output text

$PRIVATE_SUBNET_2 = aws ec2 create-subnet `
  --vpc-id $VPC_ID `
  --cidr-block "10.0.11.0/24" `
  --availability-zone "${REGION}c" `
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=private-subnet-1c}]" `
  --region $REGION `
  --profile $PROFILE `
  --query 'Subnet.SubnetId' `
  --output text
```

#### Create Internet Gateway & Route Tables

```powershell
# Create Internet Gateway
$IGW_ID = aws ec2 create-internet-gateway `
  --tag-specifications "ResourceType=internet-gateway,Tags=[{Key=Name,Value=webapp-igw}]" `
  --region $REGION `
  --profile $PROFILE `
  --query 'InternetGateway.InternetGatewayId' `
  --output text

Write-Host "IGW ID: $IGW_ID"

# Attach to VPC
aws ec2 attach-internet-gateway `
  --vpc-id $VPC_ID `
  --internet-gateway-id $IGW_ID `
  --region $REGION `
  --profile $PROFILE

# Create public route table
$PUBLIC_RT = aws ec2 create-route-table `
  --vpc-id $VPC_ID `
  --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=public-rt}]" `
  --region $REGION `
  --profile $PROFILE `
  --query 'RouteTable.RouteTableId' `
  --output text

# Add route to IGW
aws ec2 create-route `
  --route-table-id $PUBLIC_RT `
  --destination-cidr-block "0.0.0.0/0" `
  --gateway-id $IGW_ID `
  --region $REGION `
  --profile $PROFILE

# Associate public subnets with public route table
aws ec2 associate-route-table `
  --subnet-id $PUBLIC_SUBNET `
  --route-table-id $PUBLIC_RT `
  --region $REGION `
  --profile $PROFILE

aws ec2 associate-route-table `
  --subnet-id $PUBLIC_SUBNET_2 `
  --route-table-id $PUBLIC_RT `
  --region $REGION `
  --profile $PROFILE
```

---

### Step 2: Security Groups

Controls inbound/outbound traffic for ALB and EC2 instances.

#### Create ALB Security Group

```powershell
$ALB_SG = aws ec2 create-security-group `
  --group-name "webapp-alb-sg" `
  --description "Security group for Application Load Balancer" `
  --vpc-id $VPC_ID `
  --region $REGION `
  --profile $PROFILE `
  --query 'GroupId' `
  --output text

Write-Host "ALB SG: $ALB_SG"

# Allow HTTP
aws ec2 authorize-security-group-ingress `
  --group-id $ALB_SG `
  --protocol tcp `
  --port 80 `
  --cidr 0.0.0.0/0 `
  --region $REGION `
  --profile $PROFILE

# Allow HTTPS
aws ec2 authorize-security-group-ingress `
  --group-id $ALB_SG `
  --protocol tcp `
  --port 443 `
  --cidr 0.0.0.0/0 `
  --region $REGION `
  --profile $PROFILE
```

#### Create EC2 Security Group

```powershell
$EC2_SG = aws ec2 create-security-group `
  --group-name "webapp-ec2-sg" `
  --description "Security group for EC2 instances" `
  --vpc-id $VPC_ID `
  --region $REGION `
  --profile $PROFILE `
  --query 'GroupId' `
  --output text

Write-Host "EC2 SG: $EC2_SG"

# Allow traffic from ALB (port 8080 or 5000)
aws ec2 authorize-security-group-ingress `
  --group-id $EC2_SG `
  --protocol tcp `
  --port 8080 `
  --source-group $ALB_SG `
  --region $REGION `
  --profile $PROFILE

# Allow SSH (restrict to your IP in production)
aws ec2 authorize-security-group-ingress `
  --group-id $EC2_SG `
  --protocol tcp `
  --port 22 `
  --cidr 0.0.0.0/0 `
  --region $REGION `
  --profile $PROFILE
```

---

### Step 3: IAM Roles

Creates service roles for EC2 instances and deployment automation.

#### Create EC2 Instance Role

```powershell
# Create assume role policy document
$ASSUME_ROLE_POLICY = @{
    "Version" = "2012-10-17"
    "Statement" = @(
        @{
            "Effect" = "Allow"
            "Principal" = @{
                "Service" = "ec2.amazonaws.com"
            }
            "Action" = "sts:AssumeRole"
        }
    )
} | ConvertTo-Json

# Save to file
$ASSUME_ROLE_POLICY | Out-File -FilePath "assume-role-policy.json" -Encoding UTF8

# Create role
$ROLE_NAME = "webapp-ec2-role"
$ROLE_ARN = aws iam create-role `
  --role-name $ROLE_NAME `
  --assume-role-policy-document file://assume-role-policy.json `
  --profile $PROFILE `
  --query 'Role.Arn' `
  --output text

Write-Host "Role ARN: $ROLE_ARN"

# Create inline policy for CloudWatch Logs and Metrics
$CW_POLICY = @{
    "Version" = "2012-10-17"
    "Statement" = @(
        @{
            "Effect" = "Allow"
            "Action" = @(
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents",
                "logs:DescribeLogStreams"
            )
            "Resource" = "arn:aws:logs:*:*:*"
        },
        @{
            "Effect" = "Allow"
            "Action" = @(
                "cloudwatch:PutMetricData",
                "ec2:DescribeVolumes",
                "ec2:DescribeTags",
                "ec2:DescribeInstances"
            )
            "Resource" = "*"
        },
        @{
            "Effect" = "Allow"
            "Action" = @(
                "ssm:GetParameter",
                "ssm:GetParameters"
            )
            "Resource" = "arn:aws:ssm:*:*:parameter/cloudwatch-config/*"
        }
    )
} | ConvertTo-Json

$CW_POLICY | Out-File -FilePath "cw-policy.json" -Encoding UTF8

aws iam put-role-policy `
  --role-name $ROLE_NAME `
  --policy-name "CloudWatchLogsAndMetrics" `
  --policy-document file://cw-policy.json `
  --profile $PROFILE

# Create instance profile
aws iam create-instance-profile `
  --instance-profile-name "${ROLE_NAME}-profile" `
  --profile $PROFILE

aws iam add-role-to-instance-profile `
  --instance-profile-name "${ROLE_NAME}-profile" `
  --role-name $ROLE_NAME `
  --profile $PROFILE

Write-Host "Instance profile created: ${ROLE_NAME}-profile"
```

#### (Optional) Create Deployment Role for GitHub Actions

```powershell
$DEPLOY_ROLE = "webapp-github-deploy-role"

$DEPLOY_ASSUME_POLICY = @{
    "Version" = "2012-10-17"
    "Statement" = @(
        @{
            "Effect" = "Allow"
            "Principal" = @{
                "AWS" = "arn:aws:iam::ACCOUNT_ID:root"
            }
            "Action" = "sts:AssumeRole"
            "Condition" = @{
                "StringEquals" = @{
                    "sts:ExternalId" = "github-actions"
                }
            }
        }
    )
} | ConvertTo-Json

$DEPLOY_ASSUME_POLICY | Out-File -FilePath "deploy-assume-policy.json" -Encoding UTF8

aws iam create-role `
  --role-name $DEPLOY_ROLE `
  --assume-role-policy-document file://deploy-assume-policy.json `
  --profile $PROFILE

# Attach deployment permissions policy (edit as needed for your deployment tool)
```

---

### Step 4: Launch EC2 Instances

Creates instances with monitoring agent and application bootstrap.

#### Create Launch Template

```powershell
# Read user data script
$USER_DATA = Get-Content -Path "scripts/user-data.sh" -Raw
$USER_DATA_B64 = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes($USER_DATA))

$LAUNCH_TEMPLATE = @{
    "ImageId" = "ami-0c55b159cbfafe1f0"  # Amazon Linux 2 AMI (update for your region)
    "InstanceType" = "t3.micro"
    "IamInstanceProfile" = @{
        "Name" = "${ROLE_NAME}-profile"
    }
    "SecurityGroupIds" = @($EC2_SG)
    "UserData" = $USER_DATA_B64
    "TagSpecifications" = @(
        @{
            "ResourceType" = "instance"
            "Tags" = @(
                @{ "Key" = "Name"; "Value" = "webapp-instance" },
                @{ "Key" = "Environment"; "Value" = "production" }
            )
        }
    )
    "Monitoring" = @{
        "Enabled" = $true
    }
} | ConvertTo-Json -Depth 10

$LAUNCH_TEMPLATE | Out-File -FilePath "launch-template.json" -Encoding UTF8

$TEMPLATE_ID = aws ec2 create-launch-template `
  --launch-template-name "webapp-template" `
  --launch-template-data file://launch-template.json `
  --region $REGION `
  --profile $PROFILE `
  --query 'LaunchTemplate.LaunchTemplateId' `
  --output text

Write-Host "Launch Template ID: $TEMPLATE_ID"
```

#### Launch Single Instance (for testing)

```powershell
$INSTANCE_ID = aws ec2 run-instances `
  --launch-template LaunchTemplateId=$TEMPLATE_ID `
  --subnet-id $PRIVATE_SUBNET `
  --region $REGION `
  --profile $PROFILE `
  --query 'Instances[0].InstanceId' `
  --output text

Write-Host "Instance ID: $INSTANCE_ID"

# Wait for instance to be running
aws ec2 wait instance-running `
  --instance-ids $INSTANCE_ID `
  --region $REGION `
  --profile $PROFILE

# Get instance details
aws ec2 describe-instances `
  --instance-ids $INSTANCE_ID `
  --region $REGION `
  --profile $PROFILE `
  --query 'Reservations[0].Instances[0].[PrivateIpAddress,State.Name]' `
  --output table
```

---

### Step 5: Application Load Balancer (ALB)

Distributes traffic across EC2 instances.

#### Create Target Group

```powershell
$TG_NAME = "webapp-tg"
$TARGET_GROUP = aws elbv2 create-target-group `
  --name $TG_NAME `
  --protocol HTTP `
  --port 8080 `
  --vpc-id $VPC_ID `
  --health-check-protocol HTTP `
  --health-check-path "/" `
  --health-check-interval-seconds 30 `
  --health-check-timeout-seconds 5 `
  --healthy-threshold-count 2 `
  --unhealthy-threshold-count 2 `
  --region $REGION `
  --profile $PROFILE `
  --query 'TargetGroups[0].TargetGroupArn' `
  --output text

Write-Host "Target Group ARN: $TARGET_GROUP"

# Register instance with target group
aws elbv2 register-targets `
  --target-group-arn $TARGET_GROUP `
  --targets Id=$INSTANCE_ID `
  --region $REGION `
  --profile $PROFILE
```

#### Create Application Load Balancer

```powershell
$ALB_NAME = "webapp-alb"
$ALB_ARN = aws elbv2 create-load-balancer `
  --name $ALB_NAME `
  --subnets $PUBLIC_SUBNET $PUBLIC_SUBNET_2 `
  --security-groups $ALB_SG `
  --scheme internet-facing `
  --type application `
  --region $REGION `
  --profile $PROFILE `
  --query 'LoadBalancers[0].LoadBalancerArn' `
  --output text

Write-Host "ALB ARN: $ALB_ARN"

# Get ALB DNS name
$ALB_DNS = aws elbv2 describe-load-balancers `
  --load-balancer-arns $ALB_ARN `
  --region $REGION `
  --profile $PROFILE `
  --query 'LoadBalancers[0].DNSName' `
  --output text

Write-Host "ALB DNS: $ALB_DNS"

# Create listener (HTTP)
aws elbv2 create-listener `
  --load-balancer-arn $ALB_ARN `
  --protocol HTTP `
  --port 80 `
  --default-actions Type=forward,TargetGroupArn=$TARGET_GROUP `
  --region $REGION `
  --profile $PROFILE
```

---

### Step 6: Auto Scaling Group (ASG)

Automatically scales instances based on demand.

#### Create Auto Scaling Group

```powershell
$ASG_NAME = "webapp-asg"
$MIN_SIZE = 1
$MAX_SIZE = 3
$DESIRED_CAPACITY = 2

aws autoscaling create-auto-scaling-group `
  --auto-scaling-group-name $ASG_NAME `
  --launch-template LaunchTemplateId=$TEMPLATE_ID `
  --min-size $MIN_SIZE `
  --max-size $MAX_SIZE `
  --desired-capacity $DESIRED_CAPACITY `
  --default-cooldown 300 `
  --health-check-type ELB `
  --health-check-grace-period 300 `
  --vpc-zone-identifier "$PRIVATE_SUBNET,$PRIVATE_SUBNET_2" `
  --target-group-arns $TARGET_GROUP `
  --region $REGION `
  --profile $PROFILE `
  --tags "Key=Name,Value=webapp-asg-instance,PropagateAtLaunch=true" "Key=Environment,Value=production,PropagateAtLaunch=true"

Write-Host "Auto Scaling Group created: $ASG_NAME"

# Verify ASG
aws autoscaling describe-auto-scaling-groups `
  --auto-scaling-group-names $ASG_NAME `
  --region $REGION `
  --profile $PROFILE `
  --query 'AutoScalingGroups[0].[DesiredCapacity,MinSize,MaxSize,Instances[*].InstanceId]' `
  --output table
```

#### Create Scaling Policies

```powershell
# Scale up policy
$SCALE_UP_POLICY = aws autoscaling put-scaling-policy `
  --auto-scaling-group-name $ASG_NAME `
  --policy-name "scale-up-policy" `
  --policy-type TargetTrackingScaling `
  --target-tracking-configuration file:///dev/stdin <<< @"
{
  "TargetValue": 70.0,
  "PredefinedMetricSpecification": {
    "PredefinedMetricType": "ASGAverageCPUUtilization"
  },
  "ScaleOutCooldown": 60,
  "ScaleInCooldown": 300
}
"@ `
  --region $REGION `
  --profile $PROFILE `
  --query 'PolicyARN' `
  --output text

Write-Host "Scaling policy created: $SCALE_UP_POLICY"
```

---

### Step 7: CloudWatch Monitoring

Sets up metrics, log groups, and dashboards.

#### Create Log Groups

```powershell
$LOG_GROUP = "/aws/ec2/webapp"
$LOG_GROUP_ALB = "/aws/alb/webapp"

aws logs create-log-group `
  --log-group-name $LOG_GROUP `
  --region $REGION `
  --profile $PROFILE

aws logs put-retention-policy `
  --log-group-name $LOG_GROUP `
  --retention-in-days 7 `
  --region $REGION `
  --profile $PROFILE

aws logs create-log-group `
  --log-group-name $LOG_GROUP_ALB `
  --region $REGION `
  --profile $PROFILE

Write-Host "Log groups created"
```

#### Enable ALB Access Logs

```powershell
# Get AWS account ID
$ACCOUNT_ID = aws sts get-caller-identity --profile $PROFILE --query Account --output text

# ALB writes logs to this S3 bucket (create if needed)
$S3_BUCKET = "webapp-alb-logs-${ACCOUNT_ID}"

# Create S3 bucket (skip if already exists)
aws s3 mb "s3://$S3_BUCKET" --region $REGION --profile $PROFILE

# Attach bucket policy for ALB logging
$BUCKET_POLICY = @{
    "Version" = "2012-10-17"
    "Statement" = @(
        @{
            "Effect" = "Allow"
            "Principal" = @{
                "AWS" = "arn:aws:iam::127311923021:root"  # US East 1 ALB account
            }
            "Action" = "s3:PutObject"
            "Resource" = "arn:aws:s3:::${S3_BUCKET}/*"
        }
    )
} | ConvertTo-Json

$BUCKET_POLICY | Out-File -FilePath "alb-bucket-policy.json" -Encoding UTF8

aws s3api put-bucket-policy `
  --bucket $S3_BUCKET `
  --policy file://alb-bucket-policy.json `
  --profile $PROFILE

# Enable ALB access logs
aws elbv2 modify-load-balancer-attributes `
  --load-balancer-arn $ALB_ARN `
  --attributes Key=access_logs.s3.enabled,Value=true Key=access_logs.s3.bucket,Value=$S3_BUCKET `
  --region $REGION `
  --profile $PROFILE
```

#### Create CloudWatch Dashboard

```powershell
$DASHBOARD_BODY = @{
    "widgets" = @(
        @{
            "type" = "metric"
            "properties" = @{
                "metrics" = @(
                    @("AWS/EC2", "CPUUtilization", @{ "stat" = "Average" }),
                    @("AWS/ApplicationELB", "TargetResponseTime", @{ "stat" = "Average" }),
                    @("AWS/ApplicationELB", "UnHealthyHostCount", @{ "stat" = "Maximum" }),
                    @("AWS/ApplicationELB", "RequestCount", @{ "stat" = "Sum" })
                )
                "period" = 300
                "stat" = "Average"
                "region" = $REGION
                "title" = "Application Metrics"
            }
        }
    )
} | ConvertTo-Json -Depth 10

$DASHBOARD_BODY | Out-File -FilePath "dashboard-body.json" -Encoding UTF8

aws cloudwatch put-dashboard `
  --dashboard-name "webapp-dashboard" `
  --dashboard-body file://dashboard-body.json `
  --region $REGION `
  --profile $PROFILE

Write-Host "CloudWatch dashboard created"
```

---

### Step 8: SNS Alerts

Sends notifications for critical events.

#### Create SNS Topic and Subscriptions

```powershell
$TOPIC_NAME = "webapp-alerts"
$TOPIC_ARN = aws sns create-topic `
  --name $TOPIC_NAME `
  --region $REGION `
  --profile $PROFILE `
  --query 'TopicArn' `
  --output text

Write-Host "SNS Topic ARN: $TOPIC_ARN"

# Subscribe email
$EMAIL = "alerts@example.com"
aws sns subscribe `
  --topic-arn $TOPIC_ARN `
  --protocol email `
  --notification-endpoint $EMAIL `
  --region $REGION `
  --profile $PROFILE

# Subscribe SMS (optional)
$PHONE = "+1234567890"
aws sns subscribe `
  --topic-arn $TOPIC_ARN `
  --protocol sms `
  --notification-endpoint $PHONE `
  --region $REGION `
  --profile $PROFILE

Write-Host "Email subscription created. Check $EMAIL for confirmation."
```

#### Create CloudWatch Alarms

```powershell
# High CPU alarm
aws cloudwatch put-metric-alarm `
  --alarm-name "webapp-high-cpu" `
  --alarm-description "Alert when CPU > 80%" `
  --metric-name CPUUtilization `
  --namespace AWS/EC2 `
  --statistic Average `
  --period 300 `
  --threshold 80 `
  --comparison-operator GreaterThanThreshold `
  --evaluation-periods 2 `
  --alarm-actions $TOPIC_ARN `
  --region $REGION `
  --profile $PROFILE

# Unhealthy targets alarm
aws cloudwatch put-metric-alarm `
  --alarm-name "webapp-unhealthy-targets" `
  --alarm-description "Alert when unhealthy target count > 0" `
  --metric-name UnHealthyHostCount `
  --namespace AWS/ApplicationELB `
  --statistic Maximum `
  --period 60 `
  --threshold 0 `
  --comparison-operator GreaterThanThreshold `
  --evaluation-periods 1 `
  --alarm-actions $TOPIC_ARN `
  --region $REGION `
  --profile $PROFILE

Write-Host "CloudWatch alarms created"
```

---

### Step 9: CloudTrail Audit Logging

Records API calls for compliance and troubleshooting.

#### Enable CloudTrail

```powershell
# Create S3 bucket for CloudTrail logs
$TRAIL_BUCKET = "webapp-cloudtrail-${ACCOUNT_ID}"

aws s3 mb "s3://$TRAIL_BUCKET" --region $REGION --profile $PROFILE

# Attach bucket policy
$TRAIL_POLICY = @{
    "Version" = "2012-10-17"
    "Statement" = @(
        @{
            "Sid" = "AWSCloudTrailAclCheck"
            "Effect" = "Allow"
            "Principal" = @{
                "Service" = "cloudtrail.amazonaws.com"
            }
            "Action" = "s3:GetBucketAcl"
            "Resource" = "arn:aws:s3:::${TRAIL_BUCKET}"
        },
        @{
            "Sid" = "AWSCloudTrailWrite"
            "Effect" = "Allow"
            "Principal" = @{
                "Service" = "cloudtrail.amazonaws.com"
            }
            "Action" = "s3:PutObject"
            "Resource" = "arn:aws:s3:::${TRAIL_BUCKET}/*"
            "Condition" = @{
                "StringEquals" = @{
                    "s3:x-amz-acl" = "bucket-owner-full-control"
                }
            }
        }
    )
} | ConvertTo-Json

$TRAIL_POLICY | Out-File -FilePath "trail-policy.json" -Encoding UTF8

aws s3api put-bucket-policy `
  --bucket $TRAIL_BUCKET `
  --policy file://trail-policy.json `
  --profile $PROFILE

# Create CloudTrail
$TRAIL_NAME = "webapp-trail"
aws cloudtrail create-trail `
  --name $TRAIL_NAME `
  --s3-bucket-name $TRAIL_BUCKET `
  --is-multi-region-trail `
  --enable-log-file-validation `
  --region $REGION `
  --profile $PROFILE

# Start logging
aws cloudtrail start-logging `
  --trail-name $TRAIL_NAME `
  --region $REGION `
  --profile $PROFILE

Write-Host "CloudTrail enabled: $TRAIL_NAME"
```

---

### Step 10: GitHub Actions CI/CD

Automated testing, building, and deployment.

#### Create GitHub Actions Workflow

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy to AWS

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  AWS_REGION: us-east-1
  REGISTRY: ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.us-east-1.amazonaws.com

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.9'
      
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r src/requirements.txt
      
      - name: Run tests
        run: |
          cd src
          pytest -v

  deploy:
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}
          role-session-name: github-actions
      
      - name: Update Auto Scaling Group
        run: |
          # Example: force deployment via ASG update
          aws autoscaling update-auto-scaling-group \
            --auto-scaling-group-name webapp-asg \
            --desired-capacity 2 \
            --region ${{ env.AWS_REGION }}
      
      - name: Deploy application
        run: |
          # Upload to S3 or deploy via SSM Session Manager
          aws s3 sync src/ s3://webapp-deployment-bucket/ --delete
```

#### Store Secrets in GitHub

1. Go to repository **Settings → Secrets and variables → Actions**
2. Add secrets:
   - `AWS_ROLE_ARN`: `arn:aws:iam::ACCOUNT_ID:role/webapp-github-deploy-role`
   - `AWS_ACCOUNT_ID`: Your AWS account ID

---

### Step 11: Verify Complete Setup

```powershell
# 1. Check VPC
aws ec2 describe-vpcs --region $REGION --profile $PROFILE --query "Vpcs[0]"

# 2. Check instances
aws ec2 describe-instances --region $REGION --profile $PROFILE --query "Reservations[0].Instances[0].[InstanceId,State.Name,PrivateIpAddress]"

# 3. Check ALB
aws elbv2 describe-load-balancers --region $REGION --profile $PROFILE --query "LoadBalancers[0].DNSName"

# 4. Check ASG
aws autoscaling describe-auto-scaling-groups --region $REGION --profile $PROFILE --query "AutoScalingGroups[0].Instances"

# 5. Check CloudWatch Alarms
aws cloudwatch describe-alarms --region $REGION --profile $PROFILE --query "MetricAlarms[*].[AlarmName,StateValue]"

# 6. Test application via ALB
curl http://$ALB_DNS

# 7. Check CloudTrail events
aws cloudtrail lookup-events --region $REGION --profile $PROFILE --max-results 5
```

---

## Cleanup

When done, remove resources to avoid unnecessary charges:

```powershell
# Delete ASG (will terminate instances)
aws autoscaling delete-auto-scaling-group `
  --auto-scaling-group-name $ASG_NAME `
  --force-delete `
  --region $REGION `
  --profile $PROFILE

# Delete ALB
aws elbv2 delete-load-balancer `
  --load-balancer-arn $ALB_ARN `
  --region $REGION `
  --profile $PROFILE

# Delete target group
aws elbv2 delete-target-group `
  --target-group-arn $TARGET_GROUP `
  --region $REGION `
  --profile $PROFILE

# Delete VPC (detaches IGW and deletes subnets automatically)
aws ec2 delete-internet-gateway `
  --internet-gateway-id $IGW_ID `
  --region $REGION `
  --profile $PROFILE

aws ec2 delete-vpc `
  --vpc-id $VPC_ID `
  --region $REGION `
  --profile $PROFILE

# Delete IAM role
aws iam delete-instance-profile `
  --instance-profile-name "${ROLE_NAME}-profile" `
  --profile $PROFILE

aws iam delete-role-policy `
  --role-name $ROLE_NAME `
  --policy-name "CloudWatchLogsAndMetrics" `
  --profile $PROFILE

aws iam delete-role `
  --role-name $ROLE_NAME `
  --profile $PROFILE

Write-Host "Cleanup complete"
```

---

## Troubleshooting & Best Practices

| Issue | Solution |
|-------|----------|
| Instances not starting | Check IAM role permissions, user data script errors in `/var/log/cloud-init-output.log` |
| ALB shows unhealthy targets | Verify security group allows ALB → EC2 traffic, check instance port (8080 vs 5000) |
| CloudWatch metrics not appearing | Ensure EC2 instance has IAM role with CloudWatch permissions, check agent config |
| CloudTrail not logging | Verify S3 bucket policy is correct, check trail status with `aws cloudtrail describe-trails` |
| GitHub Actions deploy fails | Check AWS role trust relationship, ensure role is assumable from GitHub |
| High costs | Use `aws ce get-cost-and-usage` to identify expensive resources; consider auto-scaling down at night |

---

## Additional Resources

- [Detailed step guides](steps/): See numbered markdown files for in-depth explanations
- [Costs & Pricing](costs.md): Breakdown of expected monthly costs
- [Challenges & Solutions](challenges.md): Known issues and workarounds
- [AWS Documentation](https://docs.aws.amazon.com/): Official AWS guides
