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
