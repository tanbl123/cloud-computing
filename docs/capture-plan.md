# Build and Capture Plan

Work this alongside the 17-stage runbook. The runbook says what to build. This says what to
photograph while you build it, so the report is half-written by the time the stack is up.

Every screenshot below is named for the evidence ID it satisfies. Save it under that exact
filename. When you write the report, you drag the file in and the caption is already decided.

---

## Read this before stage 01

**Six things get destroyed by later stages.** If you miss them you cannot go back without
rebuilding, so they are marked ⚠ **LAST CHANCE** at the stage that produces them.

| What disappears | Alive during | Killed at | Evidence you lose |
|---|---|---|---|
| CapstonePOC and its local MySQL | 08 – 10 | 10 | F-1 and F-2 on a single instance, which is Phase 2 of the brief on its own, plus the pre-migration row count |
| Build-SG rules | 04 – 17 | 17 | SEC-1 proof that SSH was limited to your IP rather than open to the world |
| App-Server standalone instance | 10 – 14 | 14 | SEC-2 on one screen: IAM role attached, app on RDS, no password in code |
| NAT gateway | 03 onward | between sessions | Route table and NAT evidence goes stale if you delete it to save budget |
| The load test itself | during 16 | when it ends | HP-2 and SC-2, which only exist while load is running |
| Pre-load baseline timing | before 16 | first request | HP-2 needs a normal number to compare peak against |

### Screenshot rules that decide whether a capture counts

Include the browser or console chrome so the **region selector and the resource name are both in
frame**. A cropped panel showing a rule with no context proves nothing to a marker, and the brief
asks for evidence, not decoration.

Capture at the moment the state is interesting. A CloudWatch graph taken the next morning has the
spike compressed into two pixels.

**Never screenshot a plaintext password.** `get-secret-value` prints your RDS master password to
the terminal. If that appears in your report you have undermined the exact C5 claim the screenshot
was meant to support. Capture the ARN response from `create-secret`, and the console entry for
`Mydbsecret`, and redact anything else.

### Folder to make now

```
evidence/
  F-1-alb-landing.png        LB-1-alb-detail.png      HA-1-subnets.png
  F-2a-view.png              LB-2-targets-two-az.png  HA-2-rds-backups.png
  F-2b-add.png               SC-1-launch-template.png SEC-1-*.png
  ...                        ...                      values.md
```

`values.md` is a plain text file for things that are not screenshots but that you will need and
will forget: VPC ID, RDS endpoint, ALB DNS name, AMI ID, the POC private IP, the Cloud9 security
group ID, and your load test numbers.

---

## Stage 01 — Lab start

Nothing to capture. Record in `values.md`: today's date, and the lab budget figure showing at the
start of the session, so you can watch the burn rate across sessions.

## Stage 02 — VPC and six subnets

| Capture | Filename | Must be in frame | Report section |
|---|---|---|---|
| Subnets list filtered to asm-VPC | `HA-1-subnets.png` | All six subnets, the **AZ column**, and the names | §6.4 C4, §4.1 |
| VPC detail page | `HA-1-vpc-detail.png` | CIDR 10.0.0.0/16 and DNS hostnames enabled | §4.1 |

Record: the VPC ID.

## Stage 03 — IGW, NAT and route tables

| Capture | Filename | Must be in frame | Report section |
|---|---|---|---|
| asm-RT-Public routes and associations | `SEC-1-rt-public.png` | 0.0.0.0/0 to asm-IGW, both public subnets associated | §4.1 |
| asm-RT-App routes and associations | `SEC-1-rt-app.png` | 0.0.0.0/0 to asm-NAT, both app subnets associated | §4.1 |
| asm-RT-DB routes and associations | `SEC-1-rt-db.png` | **Only** the local route, both DB subnets associated | §6.5 C5 |

`SEC-1-rt-db.png` is the strongest single security screenshot in the whole build. It proves the
database has no path to the internet by routing, not merely by a security group rule that someone
could later widen. Make sure the Routes tab clearly shows nothing but the local entry.

Record: NAT gateway ID and its Elastic IP.

## Stage 04 — Security groups

| Capture | Filename | Must be in frame | Report section |
|---|---|---|---|
| ALB-SG inbound rules | `SEC-1-sg-alb.png` | 80 from 0.0.0.0/0 | §6.5 C5 |
| App-SG inbound rules | `SEC-1-sg-app.png` | Source column showing the **ALB-SG group ID and name**, not a CIDR | §6.5 C5 |
| DB-SG inbound rules | `SEC-1-sg-db.png` | Source column showing App-SG and the Cloud9 SG | §6.5 C5 |
| Build-SG inbound rules | `SEC-1-sg-build.png` | SSH 22 with source **My IP**, not 0.0.0.0/0 | §6.5 C5 |

⚠ **LAST CHANCE (deleted at stage 17).** `SEC-1-sg-build.png` is the only proof you will have that
administrative access was restricted to a controlled IP. C5 asks for that explicitly and Build-SG
is gone by the time you write the report.

The Source column is what separates the Good and Excellent bands on C5. If your screenshot shows an
IP range where it should show `sg-...`, you have evidence of the wrong thing.

## Stage 05 — Cloud9

| Capture | Filename | Must be in frame | Report section |
|---|---|---|---|
| Cloud9 environment detail | `HP-1-cloud9-env.png` | Name, instance type t3.micro, the VPC | §6.7 C7 method |

Record: the Cloud9 security group ID, since two rules depend on it.

## Stage 06 — RDS

| Capture | Filename | Must be in frame | Report section |
|---|---|---|---|
| Connectivity and security tab | `SEC-1-rds-private.png` | **Publicly accessible: No**, and DB-SG | §6.5 C5 |
| Maintenance and backups tab | `HA-2-rds-backups.png` | Retention period of 7 days, automated backups on | §6.4 C4 |
| DB subnet group | `HA-2-db-subnet-group.png` | Both AZs listed | §4.3 |

Record: the RDS endpoint. You need it in stages 07, 09 and 10, and it goes in the report.

## Stage 07 — Secret

| Capture | Filename | Must be in frame | Report section |
|---|---|---|---|
| create-secret CLI response | `SEC-2-secret-created.png` | The ARN in the response | §6.5 C5 |
| Secrets Manager console | `SEC-2-secret-console.png` | `Mydbsecret` listed | §6.5 C5 |

Do not capture the output of `get-secret-value` unless you redact the password first.

## Stage 08 — Phase-2 POC

| Capture | Filename | Must be in frame | Report section |
|---|---|---|---|
| App on the instance public IP | `F-1-poc-landing.png` | The public IP in the address bar | §6.1 C1 |
| Student list before changes | `F-2a-view.png` | The record list | §6.1 C1 |
| Adding a record | `F-2b-add.png` | The form, then the list showing the new row | §6.1 C1 |
| Editing a record | `F-2c-edit.png` | The changed value after a page reload | §6.1 C1 |
| Deleting a record | `F-2d-delete.png` | The list without the row | §6.1 C1 |

| Log line proving local fallback | `SEC-2a-secrets-not-found.png` | `grep "Secrets not found" /home/ubuntu/app.log` | §6.5 C5, appendix |

Launch this instance with **no IAM instance profile**. That is deliberate: the secret lookup fails,
the app falls back to local MySQL, and Phase 2 behaves as designed. The `Secrets not found` line in
the log is worth capturing, because paired with its absence at stage 10 it proves the credential
mechanism rather than merely asserting it.

⚠ **LAST CHANCE (terminated at stage 10).** This instance is Phase 2 of your brief. Add five or six
realistic records before you go on, because the migration at stage 09 is only interesting if it
moves something, and "Test1, Test2, Test3" reads badly in a report.

Reload the page after every write before you capture it. A screenshot of a submitted form proves
the form submitted, not that the database kept anything.

Record: the POC private IPv4, needed for the migration.

## Stage 09 — Migration

| Capture | Filename | Must be in frame | Report section |
|---|---|---|---|
| Row count on the POC before migrating | `F-3a-before.png` | The SELECT output on the local database | §6.1 C1 |
| The two migration commands | `F-3b-commands.png` | Both the mysqldump and the mysql import | §6.1 C1 |
| SELECT against RDS after import | `F-3c-after.png` | The **RDS endpoint visible in the command line** | §6.1 C1 |

The endpoint has to be visible in `F-3c-after.png`. Without it the screenshot shows rows in some
database, which is not the claim you are making.

## Stage 10 — App Server on RDS

| Capture | Filename | Must be in frame | Report section |
|---|---|---|---|
| Instance IAM role attachment | `SEC-2-instance-role.png` | LabInstanceProfile on the instance | §6.5 C5 |
| App running against RDS | `F-2e-app-on-rds.png` | The app working on this instance's IP | §6.1 C1 |
| Code or config reading the secret | `SEC-2-no-hardcoded-creds.png` | The secret lookup where a password would be | §6.5 C5 |
| App working after a reboot | `F-2f-after-reboot.png` | The app serving after the instance restarts | §6.1 C1 |

⚠ **LAST CHANCE (terminated at stage 14).** `SEC-2-instance-role.png` and
`SEC-2-no-hardcoded-creds.png` together are your whole credential-handling argument.

This instance **must** have `LabInstanceProfile` attached, the opposite of stage 08. Confirm the log
does **not** say `Secrets not found` before you photograph anything: that absence is what proves it
read the secret and is talking to RDS. Capture it as `SEC-2b-secret-used.png` and pair it with the
stage 08 screenshot. Two log lines, side by side, are the clearest possible evidence of decoupling.

## Stage 11 — AMI

| Capture | Filename | Must be in frame | Report section |
|---|---|---|---|
| AMI in the available state | `SC-1-ami.png` | Name and status Available | §4.2 |

Record: the AMI ID.

## Stage 12 — Launch template

| Capture | Filename | Must be in frame | Report section |
|---|---|---|---|
| Launch template summary | `SC-1-launch-template.png` | AMI, instance type, App-SG and LabInstanceProfile **together on one screen** | §6.3 C3 |

## Stage 13 — Target group and ALB

| Capture | Filename | Must be in frame | Report section |
|---|---|---|---|
| ALB detail page | `LB-1-alb-detail.png` | DNS name, scheme internet-facing, both AZs | §6.2 C2 |
| Listener rule | `LB-1-listener.png` | HTTP:80 forwarding to App-TG | §6.2 C2 |
| Target group configuration | `LB-1-target-group.png` | Protocol, port and health check path | §6.2 C2 |

Record: the ALB DNS name. This is the URL you submit and demo.

## Stage 14 — Auto Scaling group

| Capture | Filename | Must be in frame | Report section |
|---|---|---|---|
| Target group Targets tab | `LB-2-targets-two-az.png` | Two healthy targets with **different AZ values** | §6.2 C2 |
| ASG detail page | `SC-1-asg-detail.png` | Min, desired, max, and both subnets | §6.3 C3 |
| Scaling policy | `SC-1-scaling-policy.png` | Target tracking on CPU with the target value | §6.3 C3 |
| App through the load balancer | `F-1-alb-landing.png` | The **ALB DNS name in the address bar** | §6.1 C1 |

`LB-2-targets-two-az.png` is the most efficient screenshot in the build. One image proves
cross-AZ distribution for C2 and a working Auto Scaling group for C3.

`F-1-alb-landing.png` replaces the stage 08 version as your submitted public URL. Keep both. The
first shows Phase 2, this one shows the finished system.

## Stage 15 — Flow logs

| Capture | Filename | Must be in frame | Report section |
|---|---|---|---|
| Flow log configuration | `SEC-1-flowlog-config.png` | Destination and filter | §6.5 C5 |
| Captured records | `SEC-1-flowlog-records.png` | Ideally an ACCEPT on port 80 and a REJECT | §6.5 C5 |

## Stage 16 — Load test

Budget a full hour and read this section before you start, because most of it cannot be recaptured.

**Before generating any load**, load the page normally and record the response time from the
browser Network tab. Save as `HP-2-baseline.png`. Without it you have no normal to compare against
and the C7 discussion has nothing to say.

| Capture | Filename | Must be in frame | Report section |
|---|---|---|---|
| Each loadtest command and its summary | `HP-1-run-normal.png`, `HP-1-run-variable.png`, `HP-1-run-peak.png` | The command with its parameters, and the result summary | §6.7 C7 |
| CloudWatch ASG CPUUtilization | `HP-2-cpu.png` | The time range spanning the test | §6.7 C7 |
| CloudWatch ALB TargetResponseTime and RequestCount | `HP-2-latency.png` | The same window | §6.7 C7 |
| Target group HealthyHostCount | `SC-2-healthy-hosts.png` | The rise from 2 to 3 or 4 | §6.3 C3 |
| ASG Activity history | `SC-2-activity.png` | Scale-out **and** scale-in with timestamps | §6.3 C3 |

⚠ **LAST CHANCE.** Everything here is live-only. Set the CloudWatch time range while the test is
still fresh, because the default view will flatten your spike into nothing within a day.

Stop the load and wait ten to fifteen minutes for scale-in before you capture `SC-2-activity.png`.
Most groups never capture scale-in and the brief lists it explicitly.

Record in `values.md` as you go: target rps, achieved rps, mean and p95 latency, error count, and
instance count before and after, for each of the three runs. That table is §6.7 of the report and
it is much easier to fill in now than to reconstruct from screenshots later.

## Stage 17 — Lock down and cost

| Capture | Filename | Must be in frame | Report section |
|---|---|---|---|
| Pricing Calculator estimate summary | `CO-1-estimate-summary.png` | 12 months, us-east-1 | §5.1 |
| Per-service breakdown | `CO-1-estimate-detail.png` | Every service line | §5.1, appendix |

Before you delete Build-SG, confirm `SEC-1-sg-build.png` from stage 04 exists. Once it is gone you
cannot prove SSH was restricted.

**Order matters here.** Remove the Build-SG source from `DB-SG` *first*, then delete `Build-SG`.
Doing it the other way round fails with "resource sg-xxxx has a dependent object", because AWS will
not delete a group that another group's rule still references. The runbook's Harden list has these
two steps the wrong way round; its Teardown section has them right.

Recapture `SEC-1-sg-db.png` afterwards. The version showing only App-SG and Cloud9 as sources is a
stronger C5 exhibit than the one that still lists Build-SG.

Export the estimate as a file as well as screenshotting it. The report asks for the export.

---

## Evidence not produced by any stage

Three things are marked work and no AWS console will hand them to you.

**HA-1, the architecture diagram**, is drawn by you, not captured. The subnet and route table
screenshots support it but do not replace it.

**CO-2, the sizing rationale**, is writing, and the decision log in the build log already holds the
reasons. Transfer them rather than inventing new ones.

**DEMO-1, the contribution log**, is filled in by the three of you and is marked individually.
