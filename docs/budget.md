# Lab budget plan

Each lab account has US$100. This stack is expensive enough that leaving it running will end the
account before submission, and cheap enough to be safe if it is parked between sessions. The
difference between the two is roughly a factor of three, so the discipline matters more than any
sizing choice.

Prices are approximate us-east-1 list prices. Verify them in the AWS Pricing Calculator, which you
have to build anyway for evidence CO-1.

## What it costs

| State | Per day | Per week | What is running |
|---|---|---|---|
| Fully running | $3.10 | $21.70 | NAT gateway, ALB, RDS, 2 instances, 3 public IPv4 |
| Parked, ALB kept | $0.95 | $6.63 | ALB and its IPs, RDS storage, snapshots |
| Parked, ALB deleted | $0.13 | $0.89 | Storage only |

Breakdown of the fully running figure, per day: NAT gateway $1.08, load balancer $0.54, RDS
instance $0.41, two t3.micro $0.50, three public IPv4 addresses $0.36, storage and CloudWatch the
remainder. Public IPv4 has been charged since February 2024 even when the address is in use, which
is easy to overlook.

## How long $100 lasts

| Days to submission | Left running | Parked between sessions |
|---|---|---|
| 30 | $93 | $28 |
| 45 | **$139 over budget** | $43 |
| 60 | **$186 over budget** | $57 |
| 75 | **$232 over budget** | $71 |

Left running, the credit is gone in **32 days**. Parked, it lasts about **105 days**.

A realistic plan of 60 days parked plus 40 hours of active building comes to about $60, leaving
roughly $40 of headroom for the demonstration and for mistakes.

## Parking procedure, run at the end of every session

1. Auto Scaling group: set desired and minimum to 0. Instances stop costing immediately.
2. Delete the NAT gateway and release its Elastic IP. This is the single biggest line item.
3. Stop the RDS instance. Note that RDS restarts itself automatically after 7 days.
4. Leave the ALB alone. See below.

To resume: recreate the NAT gateway, re-point the `asm-RT-App` default route at it, start RDS, and
set the Auto Scaling group back to 2. About three minutes.

## Do not delete the load balancer

Deleting the ALB saves $0.82 a day and costs you the DNS name, which is your submitted public URL
and is captured in evidence F-1. A recreated load balancer gets a different name, so your report
would point at a system that no longer exists. Pay to keep it.

## One stack needs to survive, not three

Criteria C1 to C7 are shared group marks and the group demonstrates one system. So only one of the
three accounts has to keep a parked stack alive until demo day. The other two should build the
whole thing, capture their evidence, finish their individual lab work, and then delete everything,
which puts them at roughly $10 total. Whoever has the most remaining budget should be the one who
carries the surviving stack.

## Watch out for

The budget figure shown in the lab interface comes from AWS Budgets and lags real spending by 8 to
12 hours. Treat it as optimistic and park early rather than late.

Do not delete anything the day before the demonstration. Recreating a NAT gateway is quick;
debugging a half-restored stack under time pressure is not.

If a session has to be cut short, drop stage 15 (VPC flow logs). It is the only part of the build
that the AWS project brief does not require.
