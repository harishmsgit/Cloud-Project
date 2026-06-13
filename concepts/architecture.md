# Architecture — AWS Architecture

> **All architecture diagrams for this topic live in this file**, expressed as Mermaid.
> No other folder (installation, demos, labs, projects, reference) contains architecture
> diagrams — they link back here.

## Technical Explanation

This document is the visual map of every architecture taught in the topic, one diagram per
design decision: the Global Infrastructure, the Shared Responsibility split, a basic web
app, a highly available multi-AZ design, a secure VPC, a serverless API, a storage-tier
decision flow, a monitoring loop, cost-model trade-offs, and the three end-to-end
projects. Each diagram is preceded by a sentence saying what it shows and followed by the
takeaway you need when you build it.

## Layman Explanation

Think of these as the floor plans and street maps for the city you are building on AWS:
where the buildings sit, how the roads connect them, which doors are locked, and what
happens when one neighbourhood loses power. Return to the relevant picture whenever a demo,
lab, or project asks you to wire something up.

---

## 1. AWS Global Infrastructure (Regions, AZs, Edge)

This flowchart shows how AWS's physical world nests: the globe holds Regions, a Region
holds Availability Zones, and Edge Locations sit close to users worldwide.

```mermaid
flowchart TB
    GLOBE[AWS Global Infrastructure]
    subgraph R1["Region: eu-west-1 (Ireland)"]
        AZ1[AZ a]
        AZ2[AZ b]
        AZ3[AZ c]
    end
    subgraph R2["Region: us-east-1 (N. Virginia)"]
        AZ4[AZ a]
        AZ5[AZ b]
    end
    EDGE[Edge Locations<br/>CloudFront / Route 53]
    GLOBE --> R1
    GLOBE --> R2
    GLOBE --> EDGE
    EDGE -.cache near users.-> USER[End users worldwide]
```

Takeaway: pick a **Region** for latency/compliance/price, spread across its **AZs** for
resilience, and use **Edge Locations** to get content close to users. Region → AZ → Edge is
the hierarchy behind almost every other decision.

## 2. Shared Responsibility Model

This flowchart splits security duties between AWS and you across service models.

```mermaid
flowchart LR
    subgraph AWS["AWS — Security OF the cloud"]
        HW[Hardware / data centers]
        NET[Global network]
        VIRT[Virtualization / managed software]
    end
    subgraph YOU["You — Security IN the cloud"]
        IAM[IAM users, roles, policies]
        DATA[Your data + encryption]
        OS[OS patching on EC2]
        FW[Security groups / NACLs]
        APP[Application code]
    end
    AWS --> LINE[The responsibility line<br/>moves with the service]
    YOU --> LINE
```

Takeaway: AWS secures the cloud; you secure what you put in it. The more managed the
service (Lambda/S3/DynamoDB), the less falls on you — but **IAM, data, and permissions are
always yours**.

## 3. Basic Web Application (Module 2)

This flowchart traces a request through the classic three-tier web app: User → Route 53 →
Load Balancer → EC2 → RDS.

```mermaid
flowchart LR
    USER[User browser] -->|1. DNS lookup| R53[Route 53]
    R53 -->|2. returns ALB address| USER
    USER -->|3. HTTPS request| ALB[Application Load Balancer]
    ALB -->|4. forwards| EC2[EC2 web/app server]
    EC2 -->|5. query| RDS[(RDS database)]
    RDS -->|6. rows| EC2
    EC2 -->|7. HTML/JSON| USER
```

Takeaway: **DNS** finds the front door (Route 53), the **load balancer** is the front door,
**EC2** is the kitchen that does the work, and **RDS** is the pantry that holds the data.
Each tier has one job — that separation is what makes it scalable and secure.

## 4. Highly Available Architecture (Module 3)

This flowchart shows the same app made highly available across two Availability Zones with
an Auto Scaling Group and a Multi-AZ database.

```mermaid
flowchart TB
    USER[Users] --> R53[Route 53]
    R53 --> ALB[Application Load Balancer<br/>spans AZ-a and AZ-b]
    subgraph AZA["Availability Zone a"]
        E1[EC2 instance]
        E2[EC2 instance]
    end
    subgraph AZB["Availability Zone b"]
        E3[EC2 instance]
        E4[EC2 instance]
    end
    ALB --> E1
    ALB --> E2
    ALB --> E3
    ALB --> E4
    E1 & E2 & E3 & E4 --> RDSP[(RDS primary — AZ a)]
    RDSP -. sync replication .-> RDSS[(RDS standby — AZ b)]
    ASG[Auto Scaling Group] -. adds/removes .- AZA
    ASG -. adds/removes .- AZB
```

Takeaway: spread instances across **multiple AZs** behind the **ALB**, let the **Auto
Scaling Group** keep the right number running and replace failures, and run **RDS Multi-AZ**
so the database fails over to a standby. Lose an entire AZ and the site stays up.

## 5. Secure VPC Architecture (Module 4)

This flowchart shows network segmentation: public subnets for the load balancer, private
subnets for app and database, with security groups gating each hop.

```mermaid
flowchart TB
    IGW[Internet Gateway] --> ALB
    subgraph VPC["VPC 10.0.0.0/16"]
        subgraph PUB["Public subnets (AZ a + b)"]
            ALB[ALB]
            NAT[NAT Gateway]
        end
        subgraph APP["Private app subnets"]
            EC2[EC2 app servers]
        end
        subgraph DB["Private DB subnets"]
            RDS[(RDS)]
        end
    end
    ALB -->|SG: 443 from internet| EC2
    EC2 -->|SG: 3306 from app SG only| RDS
    EC2 -->|outbound updates| NAT --> IGW
```

Takeaway: only the **ALB** is exposed to the internet; **EC2** and **RDS** live in
**private subnets**. **Security groups** allow exactly one hop each (internet→ALB→app→DB),
and outbound patches go through a **NAT Gateway**. This is defense in depth in one picture.

## 6. Serverless Architecture (Module 5)

This flowchart shows the event-driven serverless chain: API Gateway → Lambda → DynamoDB,
with CloudWatch observing.

```mermaid
flowchart LR
    USER[Client] -->|HTTPS| APIGW[API Gateway]
    APIGW -->|invokes| LAMBDA[Lambda function]
    LAMBDA -->|read/write| DDB[(DynamoDB table)]
    LAMBDA -->|logs + metrics| CW[CloudWatch]
    IAM[IAM execution role] -. grants least privilege .- LAMBDA
```

Takeaway: there are **no servers to manage**. API Gateway is the door, Lambda is code that
runs only on a request and scales automatically, DynamoDB stores data with no capacity
planning, and the **IAM execution role** gives Lambda just enough permission. You pay per
request, not per hour.

## 7. Storage Decision Flow (Module 6)

This flowchart helps choose between S3, EBS, EFS, and Glacier based on access pattern.

```mermaid
flowchart TD
    START{What are you storing?} -->|Objects: files, images, backups, static site| S3CHK{Access frequency?}
    START -->|A disk for ONE EC2 instance| EBS[EBS<br/>block storage]
    START -->|Shared files for MANY instances| EFS[EFS<br/>shared file system]
    S3CHK -->|Frequent| S3[S3 Standard]
    S3CHK -->|Infrequent| S3IA[S3 Standard-IA]
    S3CHK -->|Archive, rarely read| GLACIER[S3 Glacier / Deep Archive]
```

Takeaway: **S3** for objects (and pick the tier by how often you read them), **EBS** for a
single instance's disk, **EFS** for files shared across instances, **Glacier** for cheap
long-term archive. Matching the class to the access pattern is the biggest storage cost
lever.

## 8. Monitoring & Observability Loop (Module 7)

This sequence diagram shows the detect → alarm → notify → act loop using CloudWatch,
CloudTrail, and Config.

```mermaid
sequenceDiagram
    participant RES as AWS Resources (EC2/RDS/Lambda)
    participant CW as CloudWatch (metrics + logs)
    participant AL as CloudWatch Alarm
    participant SNS as SNS Topic
    participant ENG as On-call Engineer
    RES->>CW: emit metrics & logs
    CW->>AL: threshold check (e.g. CPU above 80% for 5m)
    AL->>SNS: ALARM state, then publish
    SNS->>ENG: email / SMS / Slack
    Note over RES,CW: CloudTrail records WHO did WHAT. AWS Config records HOW config changed
```

Takeaway: **CloudWatch** watches health and fires **alarms** through **SNS** to a human or
an automation; **CloudTrail** answers "who did this?" and **AWS Config** answers "what
changed and is it compliant?". Together they are see, alert, and audit.

## 9. Cost Model Trade-offs (Module 8)

This flowchart maps workload shape to the cheapest pricing model.

```mermaid
flowchart TD
    Q{Workload shape?} -->|Steady 24/7 baseline| SP[Savings Plan / Reserved<br/>up to ~72% off]
    Q -->|Spiky / unpredictable| OD[On-Demand<br/>pay per use]
    Q -->|Interruptible batch, CI, fault-tolerant| SPOT[Spot Instances<br/>up to ~90% off]
    Q -->|Event-driven, scales to zero| SLESS[Serverless: Lambda / Fargate]
    OD --> ASG[Add Auto Scaling so you only run what you need]
```

Takeaway: never pay On-Demand for everything. Cover the **steady baseline** with Savings
Plans/RIs, run **interruptible** work on Spot, let **spiky** work scale on-demand, and use
**serverless** when traffic is bursty. Mixing models is how big bills shrink.

## 10. Project 1 — Highly Available Web Application

This flowchart is the buildable architecture for the intermediate project: Route 53 → ALB →
Auto Scaling EC2 → RDS Multi-AZ.

```mermaid
flowchart TB
    DNS[Route 53<br/>app.example.com] --> ALB[ALB across 2 AZs]
    subgraph TIER["Auto Scaling Group (min 2, max 6)"]
        W1[EC2 AZ-a]
        W2[EC2 AZ-b]
    end
    ALB --> W1
    ALB --> W2
    W1 & W2 --> RDS[(RDS Multi-AZ<br/>MySQL/Postgres)]
    CW[CloudWatch alarms] -. scale on CPU .- TIER
```

Takeaway: this is Diagram 4 made concrete and deployable — the reference HA web stack. It
survives an instance failure (ASG replaces it), an AZ failure (other AZ + RDS standby), and
a traffic spike (ASG scales out).

## 11. Project 2 — Serverless Employee Management System

This flowchart shows the advanced project: a CRUD API with API Gateway, Lambda per
operation, and a DynamoDB table.

```mermaid
flowchart LR
    UI[Client / Postman] -->|REST| GW[API Gateway<br/>/employees]
    GW -->|POST| C[Lambda create]
    GW -->|GET| R[Lambda read/list]
    GW -->|PUT| U[Lambda update]
    GW -->|DELETE| D[Lambda delete]
    C & R & U & D --> T[(DynamoDB: Employees)]
    C & R & U & D --> LOG[CloudWatch Logs]
```

Takeaway: each HTTP method maps to a small Lambda over one DynamoDB table — full CRUD with
**no servers**, automatic scaling, and per-request billing. CloudWatch Logs capture every
invocation for debugging.

## 12. Project 3 — Static Website Hosting

This flowchart shows the beginner project: S3 origin, CloudFront CDN with HTTPS, fronted by
a Route 53 custom domain.

```mermaid
flowchart LR
    USER[Visitor] -->|app.example.com| R53[Route 53]
    R53 --> CF[CloudFront<br/>HTTPS + caching at edge]
    CF -->|cache miss| S3[(S3 bucket<br/>static files)]
    ACM[ACM TLS certificate] -. enables HTTPS .- CF
    OAC[Origin Access Control] -. locks bucket to CloudFront .- S3
```

Takeaway: S3 stores the files cheaply, **CloudFront** serves them fast and over HTTPS from
edge caches, **ACM** provides the certificate, and **Origin Access Control** ensures
visitors can only reach the bucket *through* CloudFront. Cheap, global, and secure static
hosting.

## Real World Example

A SaaS company composes these diagrams into one platform: marketing site on Diagram 12
(S3+CloudFront), the application on Diagram 10 (HA web tier), background jobs and webhooks
on Diagram 6 (serverless), all inside Diagram 5 (secure VPC), watched by Diagram 8
(monitoring), and tuned by Diagram 9 (cost models). Every architecture in this topic is a
slice of that whole.

## Use Cases

- Diagrams 1–2: orientation and security baseline for any account.
- Diagrams 3–5: standard three-tier web apps from simple to secure-and-HA.
- Diagram 6: APIs, webhooks, and event processing.
- Diagrams 7–9: storage, observability, and cost decisions for any design.
- Diagrams 10–12: full buildable reference projects.

## Industry Relevance

Whiteboarding these exact pictures — "draw a highly available web app on AWS", "show me a
secure VPC", "design a serverless API" — is the core of AWS system-design interviews and
the Solutions Architect exam. Being able to draw the AZ spread, explain each security-group
hop, and justify the pricing model is precisely the skill these diagrams build.
