# Architecture diagram specification (evidence HA-1)

Everything the diagram must show, taken from what was actually built rather than from the plan.
Cross-checked against `evidence/values.md` on 1 September 2026.

The assignment brief asks for the **official AWS architecture icons**, so the recommended route is
to redraw this in Lucidchart, which is available from the Lucid (Whiteboard) link in your AWS Academy
course and ships with the AWS icon set. `architecture-diagram.svg` in this folder is an accurate
reference to work from.

## What must appear, and where

### Boundaries, outermost first
1. **AWS Cloud**
2. **Region: us-east-1** — the brief requires a single Region and the estimate is priced there
3. **VPC `asm-VPC` — 10.0.0.0/16**, DNS hostnames enabled
4. **Two Availability Zones**, `us-east-1a` and `us-east-1b`, drawn as separate columns

### Inside each Availability Zone, three subnet tiers

| Tier | us-east-1a | us-east-1b | Route table |
|---|---|---|---|
| Public | `asm-Public-A` 10.0.0.0/24 | `asm-Public-B` 10.0.1.0/24 | `asm-RT-Public` → `asm-IGW` |
| Application (private) | `asm-App-A` 10.0.2.0/24 | `asm-App-B` 10.0.3.0/24 | `asm-RT-App` → `asm-NAT` |
| Database (private) | `asm-DB-A` 10.0.4.0/24 | `asm-DB-B` 10.0.5.0/24 | `asm-RT-DB` — local only |

**Label the route table on each tier.** A subnet is public or private because of its route table, and
showing that is what distinguishes an understood diagram from a copied one.

### Components and their placement

| Component | Where | Notes |
|---|---|---|
| `asm-IGW` internet gateway | VPC edge | attached to `asm-VPC` |
| `asm-ALB` Application Load Balancer | spans both public subnets | internet-facing, HTTP:80, `ALB-SG` |
| `asm-NAT` NAT gateway | `asm-Public-A` only | Elastic IP 52.73.159.122. One NAT, not two: a cost decision |
| `asm-Cloud9` | `asm-Public-A` | t3.micro, SSH, Amazon Linux 2023. Admin and load-testing host |
| Auto Scaling group `App-ASG` | spans `asm-App-A` and `asm-App-B` | min 2, desired 2, max 4, t3.micro, `App-SG` |
| `asm-rds` | `asm-DB-A` | MySQL db.t3.micro, single AZ, not publicly accessible, `DB-SG`, 7-day backups |
| DB subnet group | spans both DB subnets | `asm-DB-B` holds no instance but is required by RDS |
| AWS Secrets Manager | **outside the VPC**, regional | holds `Mydbsecret` |
| Amazon CloudWatch | **outside the VPC**, regional | scaling metrics, and the `/vpc/asm-flowlogs` log group |
| IAM role `LabRole` via `LabInstanceProfile` | attached to the app instances | how they read the secret without stored credentials |
| AWS Shield Standard | at the ALB | automatic on every ALB, nothing to configure |
| AWS WAF | at the ALB, **dashed outline** | designed but not deployed, lab restriction. Label it as such |

**`Build-SG` does not appear.** It is deleted at stage 17 and is not part of the delivered
architecture.

`asm-DB-B` should be drawn empty but present, labelled something like "required by DB subnet group".
An examiner who notices an unused subnet and finds it explained will read that as deliberate.

### Connections to draw

| From | To | Label |
|---|---|---|
| Internet users | `asm-IGW` → `asm-ALB` | HTTP 80 |
| `asm-ALB` | app instances in both AZs | HTTP 80, `App-SG` accepts only `ALB-SG` |
| App instances | `asm-rds` | TCP 3306, `DB-SG` accepts only `App-SG` |
| App instances | `asm-NAT` → `asm-IGW` → Secrets Manager | outbound only, to fetch `Mydbsecret` |
| `asm-Cloud9` | `asm-rds` | TCP 3306, migration and verification |
| VPC | CloudWatch Logs | VPC flow logs |
| `App-ASG` / `asm-ALB` | CloudWatch | metrics driving target tracking |

**Arrow direction matters.** The app-to-NAT path is outbound only. Drawing it as bidirectional
contradicts your own C5 claim that nothing reaches the private subnets from outside.

### Three things the diagram should make obvious at a glance

1. **No path into the private subnets from the internet.** Only the ALB sits in a public subnet on
   the request path.
2. **Every tier exists in both Availability Zones**, which is the C4 argument.
3. **No password anywhere in the application.** The credential arrives from Secrets Manager at
   runtime via the instance role.

## Known limitation to note beside the diagram

The database is single-AZ, so `asm-DB-A` is the one remaining single point of failure. Multi-AZ is
blocked in the Learner Lab and is named in the report as the first production enhancement. Saying
this next to the diagram is stronger than leaving a reader to spot it.
