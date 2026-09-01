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
| VPC ID | vpc- |
| asm-Public-A subnet ID | subnet- |
| asm-Public-B subnet ID | subnet- |
| asm-App-A subnet ID | subnet- |
| asm-App-B subnet ID | subnet- |
| asm-DB-A subnet ID | subnet- |
| asm-DB-B subnet ID | subnet- |

## Stage 03, gateways

| Item | Value |
|---|---|
| Internet gateway ID | igw- |
| NAT gateway ID | nat- |
| Elastic IP | |

## Stage 04 and 05, security groups

| Group | ID |
|---|---|
| ALB-SG | sg- |
| App-SG | sg- |
| DB-SG | sg- |
| Build-SG | sg- |
| Cloud9 SG (aws-cloud9-...) | sg- |

## Stage 06 and 07, data

| Item | Value |
|---|---|
| RDS endpoint | |
| RDS master username | nodeapp |
| RDS master password | **do not write it here** — agree where the group keeps it |
| Secret ARN | arn:aws:secretsmanager: |

## Stage 08 to 11, instances

| Item | Value |
|---|---|
| CapstonePOC public IP | |
| CapstonePOC **private** IP (needed for the migration) | |
| Records added before migrating | how many: |
| App-Server public IP | |
| AMI ID | ami- |

## Stage 13, the submitted URL

| Item | Value |
|---|---|
| ALB DNS name | |

## Stage 16, load test results

| Run | Target rps | Achieved rps | Mean ms | p95 ms | Errors | Instances before | Instances after |
|---|---|---|---|---|---|---|---|
| Baseline (no load) | — | — | | | — | 2 | 2 |
| Normal | 50 | | | | | | |
| Variable | 250 | | | | | | |
| Peak | 1000 | | | | | | |

Time from load starting to a new instance serving traffic: ______

## Stage 17, cost

| Item | Value |
|---|---|
| 12-month estimate total | $ |
| Budget consumed by the end | $ |
