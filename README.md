# AWS Hands-On Projects

A collection of hands-on AWS projects designed for students to build real-world skills through guided, step-by-step exercises in their own AWS accounts.

> 🚀 **First time here? Start with [SETUP.md](SETUP.md)** — it installs every tool you need
> (AWS CLI, Python, Git, Docker, …) with steps for **Linux, macOS, and Windows**.

## Projects

### IAM & Security

| Project | Services | Description |
|---------|----------|-------------|
| [IAM Roles & Policies](./iam-roles-and-policies/README.md) | IAM, STS, Lambda, EC2, ECS | Master trust vs. permission policies by building six real-world roles: user-assumed via CLI/STS, Lambda/EC2/ECS service roles, cross-account with External ID, and GitHub OIDC federation |

### Storage & CDN

| Project | Services | Description |
|---------|----------|-------------|
| [S3 + CloudFront Static Website](./s3-cloudfront-static-website/README.md) | S3, CloudFront, OAC, IAM | Host a static site in a private S3 bucket served globally over HTTPS via CloudFront; learn origin access control, default root object, custom error pages, caching, and cache invalidation |

### Messaging & Queuing

| Project | Services | Description |
|---------|----------|-------------|
| [Event-Driven Messaging with SNS & SQS](./sqs-sns-iam-messaging/README.md) | SNS, SQS, IAM | Build a fanout messaging system using pub/sub architecture |
| [Lambda Triggered by SQS with SNS Notification](./lambda-sqs-sns-trigger/README.md) | Lambda, SQS, SNS, IAM, CloudWatch | Build a serverless order-processing pipeline: SQS triggers Lambda, Lambda publishes results to SNS |

### Lambda Series (Beginner → Advanced)

Work through these four projects in order. Each builds on the previous.

| # | Project | Services | Description |
|---|---------|----------|-------------|
| 1 | [Lambda Basics](./lambda-basics/README.md) | Lambda, IAM, CloudWatch | Deploy your first Lambda, understand handlers, invoke via Console/CLI/Boto3, and read CloudWatch Logs |
| 2 | [Lambda with S3 Event Processing](./lambda-s3-event-processing/README.md) | Lambda, S3, IAM, CloudWatch | Trigger Lambda automatically on S3 uploads; process text/CSV files and write results to a destination bucket |
| 3 | [Lambda Layers](./lambda-layers/README.md) | Lambda, IAM, CloudWatch | Package third-party libraries (`requests`, `pandas`) as reusable Lambda Layers; attach multiple layers to one function |
| 4 | [Lambda Troubleshooting & Boto3 Automation](./lambda-troubleshooting-monitoring/README.md) | Lambda, S3, EC2, SQS, CloudWatch, Log Insights | Debug 8 failure scenarios, master Log Insights queries, configure DLQs, and automate EC2/S3/SQS with Boto3 |

### ECS & Fargate Series (Beginner → Advanced)

Work through these two projects in order. Project 1 covers fundamentals; Project 2 builds a production-grade deployment.

| # | Project | Services | Description |
|---|---------|----------|-------------|
| 1 | [ECS Fargate Basics](./ecs-fargate-basics/README.md) | ECS, Fargate, ECR Public, IAM, CloudWatch | Containerize a Flask app, push to ECR Public, run as a Fargate task — understand ECS fundamentals |
| 2 | [ECS Fargate Advanced](./ecs-fargate-advanced/README.md) | ECS, Fargate, ECR Public, ALB, VPC, IAM, CloudWatch, Auto Scaling | Production-grade deployment: VPC networking, Application Load Balancer, rolling deployments, Auto Scaling, Container Insights, Docker Compose |

### Web App Architecture — Native vs. Serverless (Beginner → Advanced)

Build the **same monitored web application twice** — once on native AWS, once serverless —
to compare cost, operations, scaling, and observability. Each is a complete, end-to-end
cloud architecture with monitoring, alerting, audit, and a GitHub Actions CI/CD pipeline.

| # | Project | Services | Description |
|---|---------|----------|-------------|
| 1 | [EC2 + VPC Monitored Web App](./ec2-vpc-monitored-webapp/README.md) | EC2, VPC, ALB, Auto Scaling, CloudWatch, SNS, CloudTrail, IAM, SSM, GitHub Actions | Full native architecture: custom VPC (public/private subnets, NAT), EC2 fleet behind an ALB with Auto Scaling, CloudWatch metrics/alarms/dashboards, SNS email alerts, CloudTrail audit, Boto3 monitoring automation, and an OIDC→SSM deploy pipeline |
| 2 | [Serverless Monitored Web App](./serverless-monitored-webapp/README.md) | API Gateway, Lambda, CloudWatch, SNS, CloudTrail, IAM, GitHub Actions | The same app with no servers: API Gateway + Lambda, the same CloudWatch/SNS/CloudTrail observability, Boto3 automation, and an OIDC→`update-function-code` pipeline — with a side-by-side native-vs-serverless comparison |

---

## How This Repo Is Organized

Each project lives in its own directory and follows a consistent layout:

```
project-name/
├── README.md           # Architecture overview and what you'll build
├── Dockerfile          # (container projects) How to build the image
├── docker-compose.yml  # (advanced container projects) Local dev setup
├── steps/              # Numbered, sequential step files
│   ├── 01-*.md
│   ├── 02-*.md
│   └── ...
├── src/                # Application source code
├── troubleshooting.md  # Common errors and fixes
└── challenges.md       # Extra challenges to deepen understanding
```

## Prerequisites (All Projects)

- An active AWS account with console access
- A user or role with sufficient permissions (each project specifies exactly what's needed)
- Basic familiarity with the AWS Management Console
- The required tooling installed on your laptop — see **[SETUP.md](SETUP.md)** for
  AWS CLI, Python, Git, Docker, and more, with **Linux / macOS / Windows** instructions

## Contributing a New Project

1. Create a new directory using kebab-case naming (e.g., `s3-static-website`)
2. Follow the directory structure above
3. Add an entry to the table in this README
4. Keep steps atomic — one concept per file
