# Architecture diagram specification (evidence HA-1)

Everything the diagram must show, taken from what was actually built rather than from the plan.
Cross-checked against `evidence/values.md` on 1 September 2026, and updated 5 Sep 2026 to add the
three counted additional features (stages 19–21) and correct the flow logs destination (stage 15,
issue I-07). If you are redrawing this for the report, this version — not the original — is current.

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
| AWS Secrets Manager | **outside the VPC**, regional | holds `Mydbsecret` — `host` field now points at `Data-Server`'s private IP, not `asm-rds` directly |
| Amazon CloudWatch | **outside the VPC**, regional | scaling metrics only (ASG CPU, ALB request count/latency, `ReplicaLag`) — **not** flow logs, see the corrected row below |
| Amazon S3 | **outside the VPC**, regional | two separate buckets: `asm-flowlogs-<account-id>` (VPC flow log delivery, since CloudWatch Logs delivery is blocked for this Lab's role — issue I-07) and `asm-db-backups-<account-id>` (versioned `mysqldump` backups with a lifecycle rule, stage 19) |
| IAM role `LabRole` via `LabInstanceProfile` | attached to the app instances **and** `Data-Server` | how they read the secret without stored credentials |
| AWS Shield Standard | at the ALB | automatic on every ALB, nothing to configure |
| AWS WAF | at the ALB, **dashed outline** | designed but not deployed, lab restriction. Label it as such |
| `Data-SG` security group + `Data-Server` EC2 | inside `asm-App-A` (same subnet as the app instances, not a new subnet tier) | t3.micro, `LabInstanceProfile` for Session Manager access (no SSH), runs `socat` as a persistent systemd service forwarding TCP 3306 to `asm-rds`. **Draw outside the `App-ASG` bracket** — it is a standalone instance, not part of the Auto Scaling group, and has no self-healing of its own |
| `asm-rds-replica` | `asm-DB-A`, alongside `asm-rds` | db.t3.micro, Single-AZ, read replica of `asm-rds`, same region (stage 21) |

**`Build-SG` does not appear.** It is deleted at stage 17 and is not part of the delivered
architecture.

`asm-DB-B` should be drawn empty but present, labelled something like "required by DB subnet group".
An examiner who notices an unused subnet and finds it explained will read that as deliberate. The read
replica does not change this — it is also single-AZ, in `asm-DB-A` alongside the primary.

### Connections to draw

| From | To | Label |
|---|---|---|
| Internet users | `asm-IGW` → `asm-ALB` | HTTP 80 |
| `asm-ALB` | app instances in both AZs | HTTP 80, `App-SG` accepts only `ALB-SG` |
| App instances | `Data-Server` | TCP 3306, `Data-SG` accepts only `App-SG` — **this replaces the old direct App-to-RDS arrow** |
| `Data-Server` | `asm-rds` | TCP 3306 (via `socat`), `DB-SG` now accepts only `Data-SG`, not `App-SG` directly |
| `asm-rds` | `asm-rds-replica` | asynchronous replication (not application traffic — draw distinctly, e.g. a different line style) |
| App instances | `asm-NAT` → `asm-IGW` → Secrets Manager | outbound only, to fetch `Mydbsecret` |
| `asm-Cloud9` | `asm-rds` | TCP 3306, migration and verification |
| VPC | Amazon S3 (`asm-flowlogs-*`) | VPC flow logs — corrected destination, was shown as CloudWatch Logs, now S3 (I-07) |
| `App-ASG` / `asm-ALB` / `asm-rds-replica` | CloudWatch | metrics driving target tracking, plus `ReplicaLag` |
| Cloud9 (or app tier) | S3 (`asm-db-backups-*`) | `mysqldump` backup upload, versioned, lifecycle to Glacier then deletion |

**Arrow direction matters.** The app-to-NAT path is outbound only. Drawing it as bidirectional
contradicts your own C5 claim that nothing reaches the private subnets from outside. The replication
arrow (`asm-rds` → `asm-rds-replica`) is one-way and internal to RDS — do not draw it the same way as
application traffic.

### Three things the diagram should make obvious at a glance

1. **No path into the private subnets from the internet.** Only the ALB sits in a public subnet on
   the request path.
2. **Every tier exists in both Availability Zones**, which is the C4 argument.
3. **No password anywhere in the application.** The credential arrives from Secrets Manager at
   runtime via the instance role.

## Known limitations to note beside the diagram

The database is single-AZ, so `asm-DB-A` is the one remaining single point of failure — the read
replica added at stage 21 helps with read scaling, but it is also single-AZ and does not remove this
point of failure by itself. Multi-AZ is blocked in the Learner Lab and is named in the report as the
first production enhancement. Saying this next to the diagram is stronger than leaving a reader to
spot it.

`Data-Server`, the middle-tier proxy added at stage 20, is a standalone EC2 instance, not part of an
Auto Scaling group. Unlike the app tier, it has no automatic self-healing: if it stops (as happened
when a Learner Lab session ended), nothing detects or restarts it. Worth naming as a deliberate,
honestly-reported trade-off of this additional feature, alongside the fix applied (a systemd service
so its `socat` forwarder survives a reboot, even though the instance itself must still be started
manually).
