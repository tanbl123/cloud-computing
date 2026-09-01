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
| Default SG of asm-VPC (unused) | `sg-0f6a3c8fefe61cf41` |

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
