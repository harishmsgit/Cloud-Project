# Core Concepts — AWS Architecture

## What

This document explains the **design concepts** every AWS architecture is judged on:
**High Availability, Fault Tolerance, Scalability/Elasticity, Security, and Cost
Optimization** — plus the service categories that implement them (compute, networking,
load balancing, data, serverless, storage, monitoring). Where [`fundamentals.md`](fundamentals.md)
covers *what AWS is*, this covers *the ideas you combine to design well*.

## Why

A list of services is not an architecture. The same services can produce a fragile, costly
system or a resilient, cheap one — the difference is whether you applied these concepts.
They are the reusable principles that let you reason about any design and answer the
questions an interviewer or a production incident will ask: *What happens when this fails?
What happens when traffic triples? Who can reach this? What does it cost?*

## Concept 1 — High Availability (HA)

**Technical:** High availability means the system keeps serving requests despite component
failures, measured as **uptime %** (e.g. 99.9% ≈ 8.7 hours of downtime/year; 99.99% ≈ 52
minutes). On AWS you achieve it by removing **single points of failure**: run redundant
components across **multiple Availability Zones**, put them behind a **load balancer** with
**health checks**, and use managed multi-AZ services (RDS Multi-AZ, S3, DynamoDB) that
replicate for you.

**Layman:** HA is having a spare tyre *and* knowing how to swap it fast. One server is one
tyre — a puncture strands you. Several servers across several buildings means a puncture
barely slows you down.

**Common mistake:** Putting two servers in the **same** AZ. That survives a server crash
but not a data-center outage — true HA needs **multiple AZs**.

## Concept 2 — Fault Tolerance

**Technical:** Fault tolerance is the system's ability to keep operating **with no loss of
service** when a component fails — a stronger property than HA. HA minimizes downtime;
fault tolerance aims for *zero* perceived downtime. It's achieved with redundancy +
automatic failover + health-based traffic shifting: an **ALB** stops routing to an
unhealthy instance, an **Auto Scaling Group** replaces it, **RDS Multi-AZ** fails over to a
standby, and **DynamoDB/S3** transparently survive AZ loss.

**Layman:** Fault tolerance is a plane with multiple engines: one engine quits and the
flight continues without the passengers noticing. HA is being able to land and fix the
engine quickly; fault tolerance is not needing to land at all.

**HA vs fault tolerance:**

| | High Availability | Fault Tolerance |
| --- | --- | --- |
| Goal | Minimize downtime | Zero perceived downtime |
| On failure | Brief disruption, fast recovery | No disruption |
| Cost | Lower | Higher (more redundancy) |
| Example | Multi-AZ with quick failover | Active-active across AZs/Regions |

## Concept 3 — Scalability & Elasticity

**Technical:** **Scalability** is the ability to handle more load by adding resources.
**Vertical scaling** (scale up) means a bigger instance; **horizontal scaling** (scale out)
means more instances. **Elasticity** is automatic scaling *both ways* in response to
demand. On AWS, **Auto Scaling Groups** add/remove EC2 based on metrics (CPU, request
count), **serverless** (Lambda, DynamoDB on-demand, S3) scales automatically with no
servers to manage, and load balancers spread traffic across the fleet.

**Layman:** Scalability is a restaurant that can serve more diners. Vertical = buy a bigger
oven. Horizontal = add more ovens and cooks. Elasticity = automatically calling in extra
cooks at lunchtime and sending them home in the afternoon, so you never overpay or
under-serve.

**Common mistake:** Only scaling *up* (vertical) — there's a ceiling and it's a single
point of failure. Prefer scaling *out* (horizontal) for both scale and HA.

## Concept 4 — Security

**Technical:** Security on AWS is **defense in depth** across layers: **IAM**
(least-privilege identities, roles over long-lived keys), **network** (VPC, private
subnets, **security groups** as stateful instance firewalls, **NACLs** as stateless subnet
firewalls), **data** (encryption at rest with **KMS**, in transit with TLS, secrets in
**Secrets Manager**), and **edge** (**WAF**, **Shield**). The guiding principle is
**least privilege**: grant only the permissions actually needed.

**Layman:** Security is a building with layered protection: ID cards at the door (IAM),
guards checking who goes between floors (security groups/NACLs), a vault for valuables
(encryption + Secrets Manager), and a guard at the street entrance (WAF/Shield). You don't
give every employee a master key — only the doors they need.

**Common mistake:** Using the **root account** for daily work, or attaching
`AdministratorAccess` to everything. Both violate least privilege.

## Concept 5 — Cost Optimization

**Technical:** Cost optimization is paying only for value delivered. Levers:
**right-sizing** (match instance size to real usage), **pricing models** (On-Demand,
**Reserved Instances**, **Savings Plans**, **Spot** — up to ~90% off for interruptible
work), **elasticity** (turn things off when idle; scale to zero with serverless),
**storage tiering** (S3 Standard → Infrequent Access → Glacier), and **managed services**
(often cheaper than self-running). Tag resources and watch **Cost Explorer** and
**Budgets**.

**Layman:** Cost optimization is not leaving the lights and the tap running in empty rooms.
Rent the right-sized flat (right-sizing), sign a lease for a discount if you'll stay
(Reserved/Savings Plans), grab cheap last-minute seats for flexible trips (Spot), and move
old boxes to a cheap storage unit instead of prime real estate (S3 tiering).

**Common mistake:** Leaving idle EC2, unattached EBS volumes, and old snapshots running —
they bill 24/7 whether used or not.

## Concept 6 — The Service Categories You Compose

| Category | Core services | Used for |
| --- | --- | --- |
| **Networking** | VPC, Subnets, Route 53, Internet/NAT Gateway | Private networks, DNS, internet access |
| **Edge / CDN** | CloudFront, Global Accelerator | Fast global delivery, DDoS absorption |
| **Load balancing** | ALB (HTTP), NLB (TCP/UDP), GWLB | Spread traffic, health-based routing |
| **Compute** | EC2, Auto Scaling, Lambda, ECS/EKS/Fargate | Run code (VMs, functions, containers) |
| **Relational data** | RDS, Aurora | SQL databases with transactions |
| **NoSQL data** | DynamoDB | Key-value/document at any scale |
| **Caching** | ElastiCache (Redis/Memcached) | Sub-millisecond reads, offload DBs |
| **Object storage** | S3, Glacier | Files, backups, static sites, data lakes |
| **Block / file storage** | EBS, EFS | Disks for EC2, shared file systems |
| **Security** | IAM, KMS, Secrets Manager, WAF, Shield | Identity, encryption, secrets, protection |
| **Observability** | CloudWatch, CloudTrail, Config | Metrics/logs/alarms, audit, compliance |
| **IaC / automation** | CloudFormation, CDK, Terraform | Repeatable, version-controlled infra |

## Concept 7 — Event-Driven & Serverless

**Technical:** In an **event-driven** architecture, components react to **events** (an HTTP
call, a file upload, a queue message) rather than polling. **Serverless** means no servers
to provision or patch — AWS runs the code on demand and bills per use. The canonical chain
is **API Gateway → Lambda → DynamoDB**: a request triggers a function that runs only while
needed and scales automatically. Glue services include **SQS** (queues), **SNS** (pub/sub),
and **EventBridge** (event bus).

**Layman:** Instead of keeping a chef standing in the kitchen all day (a running server),
serverless is a chef who appears the instant an order arrives, cooks it, and vanishes — and
you pay only for the minutes they cooked. When 100 orders arrive at once, 100 chefs appear.

**Common mistake:** Forcing serverless onto steady, high-volume, long-running workloads
where always-on compute is cheaper. Serverless shines for spiky, event-driven traffic.

## Real World Example

A ticketing site applies all six concepts at once. **HA + fault tolerance:** web servers in
three AZs behind an ALB, RDS Multi-AZ. **Scalability/elasticity:** an Auto Scaling Group
grows from 4 to 40 instances when a concert goes on sale, then shrinks. **Security:**
private subnets for the database, security groups that only let the ALB reach the web tier
and the web tier reach the DB, secrets in Secrets Manager, TLS everywhere. **Cost:** a
Savings Plan covers the steady baseline, Spot instances handle the surge cheaply, old
booking records tier to S3 Glacier. **Event-driven:** confirmation emails are sent by a
Lambda triggered off an SQS queue, so a flood of bookings never blocks checkout.

## Use Cases

- Designing any production web/mobile backend that must stay up and scale.
- Choosing between EC2, containers, and serverless for a given workload.
- Picking the right database (relational vs NoSQL vs cache) and storage class.
- Defending an architecture in a design review against the five pillars.

## Industry Relevance

These concepts are the daily language of cloud engineering and the backbone of the AWS
Solutions Architect certification and system-design interviews. "Make this highly
available", "scale this for a launch", "tighten the security", and "cut this bill" are
literal job tickets. Mastering the trade-offs between them — HA vs cost, serverless vs EC2,
vertical vs horizontal — is what separates someone who lists services from someone who can
architect.