# Build values

Live record for this build. Paste values into the chat as you go and they get written here, so
nobody has to fight the GitHub web editor. The blank template is in `docs/values-template.md` if a
teammate wants their own copy.

## Session

| Item | Value |
|---|---|
| Your name | |
| AWS account ID | 401858547100 |
| Lab budget at first session | $0 used of $100, 1 Sep 2026 |
| Budget now / date checked | $ ______ on ______ |

## Stage 02, network

| Item | Value |
|---|---|
| VPC ID | `vpc-0e5d12a859efbf782` |
| Main route table (auto-created) | `rtb-0da6a850010e005a2` |
| DNS hostnames | Enabled |
| AZ mapping for this account | us-east-1a = use1-az2, us-east-1b = use1-az4 (differs per account, teammates will see other numbers) |
| asm-Public-A subnet ID | `subnet-0cb19ed319b5dfa45` (10.0.0.0/24) |
| asm-Public-B subnet ID | `subnet-089126fbf226b1bc3` (10.0.1.0/24) |
| asm-App-A subnet ID | `subnet-064676218e926c74c` (10.0.2.0/24) |
| asm-App-B subnet ID | `subnet-0fb96b33f55918639` (10.0.3.0/24) |
| asm-DB-A subnet ID | `subnet-0bd19aa5bb9b8e6be` (10.0.4.0/24) |
| asm-DB-B subnet ID | `subnet-07332949cb37af58c` (10.0.5.0/24) |

## Stage 03, gateways and route tables

| Item | Value |
|---|---|
| asm-RT-Public | `rtb-0dfa185316f397b83` (asm-Public-A, asm-Public-B) |
| asm-RT-App | `rtb-076d8e7e28263d072` (asm-App-A, asm-App-B) |
| asm-RT-DB | `rtb-0df413343feabd360` (asm-DB-A, asm-DB-B) |
| Internet gateway ID | `igw-02e886e8c91c2322d` (Attached) |
| NAT gateway ID | `nat-01ff1582e8b77e903` (Available, in asm-Public-A, private 10.0.0.71) |
| Elastic IP (NAT public) | 52.73.159.122 |
| NAT created | 1 Sep 2026 16:25 GMT+8 — billing starts here |

## Stage 04 and 05, security groups

| Group | ID |
|---|---|
| ALB-SG | `sg-0dd8a35db49344985` |
| App-SG | `sg-078027c0af8c5cc2e` |
| DB-SG | `sg-0876c9dc85bd6f67f` |
| Build-SG | `sg-05d99a3d8e3b95da7` (My IP at build time: 27.125.245.206/32) |
| Cloud9 SG (`aws-cloud9-asm-Cloud9-...InstanceSecurityGroup-y8ZVJVV6DHLG`) | `sg-03363bdb3bb195521` |
| Cloud9 placement | `asm-Public-A` (subnet-0cb19ed319b5dfa45), SSH connection, Amazon Linux 2023 — draw it here on the diagram |
| Default SG of asm-VPC (unused) | `sg-0f6a3c8fefe61cf41` |

## Stage 06 and 07, data

| Item | Value |
|---|---|
| RDS endpoint | `asm-rds.ch9e5pk57w5b.us-east-1.rds.amazonaws.com` port 3306 |
| RDS master username | nodeapp |
| RDS master password | **do not write it here** — agree where the group keeps it |
| RDS instance | `asm-rds` · db.t3.micro · MySQL Community · us-east-1a · encryption at rest on |
| Secret ARN | `arn:aws:secretsmanager:us-east-1:401858547100:secret:Mydbsecret-xenf8e` |
| Secret created | 2 Sep 2026 08:29 UTC |

## Stage 08 to 11, instances

> **Public IPs change between lab sessions.** Ending the lab, or letting the timer run out, stops every
> EC2 instance. They start again with new public IPv4 addresses. The values below are the ones that were
> live when the figures were captured, so treat them as a record of that session rather than as
> addresses to browse to today. Nothing breaks, because every security group rule references another
> security group rather than an address. Private IPs, instance IDs, the RDS endpoint, the secret ARN,
> the AMI ID and the launch template ID are all stable.

| Item | Value |
|---|---|
| CapstonePOC public IP | 52.206.157.249 |
| CapstonePOC **private** IP (needed for the migration) | **10.0.0.184** |
| CapstonePOC instance ID | `i-06968508821a04f04` (no IAM role, IMDSv2 optional, Ubuntu 24.04 LTS) |
| Records added before migrating | how many: |
| App-Server instance ID | `i-0636b1e60f111d310` (IAM role LabRole via LabInstanceProfile, IMDSv2 optional, Ubuntu 24.04 LTS) |
| App-Server public IP | 54.227.135.18 |
| App-Server private IP | 10.0.0.120 |
| App-Server placement | `asm-Public-A` (subnet-0cb19ed319b5dfa45), Build-SG, launched 3 Sep 2026, **terminated 4 Sep 2026** after stage 16 |
| AMI ID (stage 11) | `ami-0d28ed0cb650c6cf4` (App-Server-AMI-v1, Available 3 Sep 2026 15:59 GMT+8, source AMI `ami-0f8a61b66d1accaee`) |

## Stage 12, launch template

| Item | Value |
|---|---|
| Launch template | `App-LT-v1` = `lt-06c91acc74c3b008f`, version 1 default, created 3 Sep 2026 |
| Contents | AMI `ami-0d28ed0cb650c6cf4`, t3.micro, vockey, App-SG `sg-078027c0af8c5cc2e`, LabInstanceProfile, metadata V1 and V2 optional, no subnet, no user data |

## Stage 13, the submitted URL

| Item | Value |
|---|---|
| **ALB DNS name (the submitted URL)** | `asm-ALB-743581282.us-east-1.elb.amazonaws.com` |
| ALB | `asm-ALB`, internet-facing, IPv4, HTTP:80 listener forwarding to App-TG, ALB-SG only, in asm-Public-A and asm-Public-B, created 3 Sep 2026 20:38 GMT+8 |
| Target group | `App-TG`, HTTP:80, HTTP1, target type Instance, health check `/` every 30s, success 200, in asm-VPC |
| Target group ARN | `arn:aws:elasticloadbalancing:us-east-1:401858547100:targetgroup/App-TG/68dfde1ef7d02f85` |

Unlike an instance public IP, the ALB DNS name is stable across lab sessions. It survives the stack
being stopped and started, so this is the address to put in the report and use for the video.

## Stage 14, Auto Scaling group

| Item | Value |
|---|---|
| Auto Scaling group | `App-ASG`, desired/min/max 2/2/4, target tracking on average CPU at 50%, 300s warmup, scale-in enabled |
| ASG instances | `i-00c957897434a961` (us-east-1a, asm-App-A) and `i-0af80d429a31ff2a2` (us-east-1b, asm-App-B), tag Name=App-Instance |
| First healthy check | 3 Sep 2026, both targets Healthy on port 80 in App-TG shortly after creation |

## Stage 15, flow logs

| Item | Value |
|---|---|
| Flow log | `asm-flowlogs` = `fl-032c44592085e4ee5`, filter All, destination S3 (not CloudWatch, see build log I-07) |
| S3 bucket | `asm-flowlogs-401858547100` |

## Stage 16, load test results

| Run | Target rps | Achieved rps | Mean ms | p95 ms | Errors | Instances before | Instances after |
|---|---|---|---|---|---|---|---|
| Baseline (no load) | — | — | 359.87 (TTFB) / 622.27 (total, cold connection) | | — | 2 | 2 |
| Normal | 50 | 50 | 42.9 | 187 | 0 | 2 | 2 |
| Variable | 250 | 247 | 5253.2 | 13092 | 14938 (33.6%) | 2 | see below (3rd attempt; degradation trend across all 3 attempts: 577.7ms/0.8% -> 3957.9ms/14.6% -> 5253.2ms/33.6%) |
| Peak | 1000 | 910 | 16140 | 30009 | 101074 (61.7%, mostly connection-level failures, not HTTP errors) | see note | 4 (ceiling reached, redo run notably worse than the first attempt: 4002.9ms mean / 50.4% errors) |

CloudWatch peaks for the redo run: ALB Target Response Time peaked at 13.7 sec (vs 5.4 sec first attempt),
35.86K total requests. ASG CPUUtilization peaked at 98.34% (vs 96.68% first attempt).

Activity history, redo run (full sequence, App-ASG settled at 4/4 Healthy afterwards):
- Before any load: i-0c29df91de914926d replaced by i-052957d0b5b7a7829 after an EC2 health check found it
  "terminated or stopped" (the I-08 Learner Lab session-stop pattern, unrelated to load).
- 06:49 PM: i-0c7479c687bb3569d failed an ELB health check, replaced by i-078fbba2b3c6488c9.
- 06:53 PM: i-052957d0b5b7a7829 (the pre-test replacement) itself failed and was replaced by
  i-03f35cdd89c04a531.
- 07:01 PM: i-078fbba2b3c6488c9 failed a second time and was replaced by i-0db87ff77fa55a1c8.
- Separately, target tracking grew desired capacity from 2 straight to 4 in one step (not 2-to-3-to-4 as
  in the first attempt) via the TargetTracking-App-ASG-AlarmHigh alarm at 10:58:25Z, launching
  i-013d6789292b02d52 and i-0bd60a44d0f58b7b4.

Two of the four running instance slots failed and were replaced twice each during this run, versus at
most once each in the first attempt — the most plausible explanation for why this run's latency and
error rate came out substantially worse.

Time from load starting to a new instance serving traffic:
- First attempt: ~5 minutes 5 seconds from alarm to InService, in two steps (18:54:40Z to 18:59:45Z for
  2-to-3, 18:56:36Z to 19:01:41Z for 3-to-4), closely matching the configured 300-second warmup.
- Redo: ~5 minutes 7 seconds from alarm to InService, in a single 2-to-4 step (10:58:34Z onward).

Scale-in: on both the first attempt and the redo, the target-tracking policy did not trigger automatic
scale-in within the available observation window despite CPU returning to baseline. Desired capacity was
manually reduced from 4 to 2 each time to demonstrate the scale-in mechanism:
- First attempt, 2026-09-03T21:16:06Z: terminated i-0a36b41e27175b3e9 and i-057f2818dadba42e6.
- Redo, 2026-09-04T12:28:46Z: terminated i-013d6789292b02d52 and i-0bd60a44d0f58b7b4.
Needing the same manual intervention on both runs supports treating the scale-in delay as a genuine,
repeatable characteristic of this policy configuration rather than a one-off anomaly.

## Stage 17, cost

| Item | Value |
|---|---|
| 12-month estimate total | $ |
| Budget consumed by the end | $ |
