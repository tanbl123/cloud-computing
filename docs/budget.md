# Lab budget plan

**Conclusion first: the budget is not a constraint on this project. Do not spend time managing it.**

Report is due **Sunday 6 September 2026**, and one week is reserved after that for the video
recording, so the stack must stay alive until about **Sunday 13 September**. That is twelve days
from 1 September. Each student has their own separate $100, so there is no shared pool to ration.

| Approach | 12-day cost | Spare from $100 |
|---|---|---|
| Leave everything running | **$37** | $63 |
| Park between sessions | $11 | $89 |

Parking saves $26 you do not need. It costs three minutes of teardown and rebuild per session, and
every rebuild is a chance to forget re-pointing the `asm-RT-App` route at the new NAT gateway,
which produces instances that boot fine and then fail to reach Secrets Manager. **Leave it
running.** Spend the saved attention on the deadline instead.

Delete everything after the video is recorded, not before.

## If the timeline slips

The figures below are what matters only if this project runs long, for example if the demo is
deferred or a resubmission is required.

| State | Per day | Per week |
|---|---|---|
| Fully running | $3.10 | $21.70 |
| Parked, ALB kept | $0.95 | $6.63 |
| Parked, ALB deleted | $0.13 | $0.89 |

Left running continuously the credit lasts **32 days**. Parked it lasts about 105 days. Breakdown
of the running figure, per day: NAT gateway $1.08, load balancer $0.54, RDS instance $0.41, two
t3.micro $0.50, three public IPv4 $0.36, storage and CloudWatch the rest. Public IPv4 has been
charged since February 2024 even when in use, which is easy to overlook.

### Parking procedure, only if you need it

1. Auto Scaling group: desired and minimum to 0.
2. Delete the NAT gateway and release its Elastic IP.
3. Stop the RDS instance. It restarts itself after 7 days.
4. Leave the ALB alone.

To resume: recreate the NAT gateway, **re-point the `asm-RT-App` default route at the new one**,
start RDS, set the Auto Scaling group back to 2.

## Do not delete the load balancer

Deleting the ALB saves $0.82 a day and costs you the DNS name, which is your submitted public URL
and is captured in evidence F-1. A recreated load balancer gets a different name, so your report
would point at a system that no longer exists. Pay to keep it.

## One stack needs to survive, not three

Criteria C1 to C7 are shared group marks and the group demonstrates one system, so only one account
has to carry a stack through to the video recording. With the budget this comfortable that is a
time argument rather than a cost one: see `schedule.md`.

## Watch out for

The budget figure shown in the lab interface comes from AWS Budgets and lags real spending by 8 to
12 hours. Treat it as optimistic and park early rather than late.

Do not delete anything the day before the demonstration. Recreating a NAT gateway is quick;
debugging a half-restored stack under time pressure is not.

If a session has to be cut short, drop stage 15 (VPC flow logs). It is the only part of the build
that the AWS project brief does not require.
