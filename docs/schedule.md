# Schedule

| Milestone | Date |
|---|---|
| Today | Tue 1 September 2026 |
| Report submission | **Sun 6 September 2026** |
| Video recording done by | Sun 13 September 2026 |
| Delete all AWS resources | after the video, not before |

**Five days to build seventeen stages and write the report.** Budget is not a constraint: each
student has their own $100, and twelve days of everything running costs about $37. Time is the
only thing in short supply, so every decision below trades money for time.

## Decisions that follow from the deadline

**Leave the stack running.** Do not park between sessions. It saves $26 nobody needs and costs
rebuild time plus the risk of forgetting the `asm-RT-App` route after recreating the NAT gateway.

**Follow the runbook as written.** All seventeen stages, no steps skipped. Stage 15 is beyond what
the AWS project brief strictly requires, but it is quick and it earns C5 material, so it stays.

**Run stage 15 before stage 16, not after.** With flow logs already collecting, the load test
generates the traffic that fills them, so `SEC-1-flowlog-records.png` shows real ACCEPTs on port 80
to your app instances instead of a nearly empty log group. Same two stages, better evidence.

**Do not all three build in parallel first.** Nothing in the rubric requires three stacks. C1 to C7
are shared group marks on one demonstrated system, and the individual D3 marks are for the guided
and challenge labs, which are separate coursework. Three simultaneous first-time builds means three
people hitting the same problem at the same time with nobody free to write the report.

## Suggested split

**Builder.** Works the runbook end to end starting today. This account's stack is the one that gets
screenshotted, demonstrated and kept alive to the recording. Whoever is most comfortable in the
console takes this.

**Second builder.** Starts a day behind, following the same runbook and the build log's agreed
values. Acts as insurance if the first build hits something unrecoverable, and gives a second
person hands-on familiarity for the demo questions.

**Writer.** Does not touch the console at first. Produces the architecture diagram (HA-1) and the
12-month Pricing Calculator estimate (CO-1 and CO-2), neither of which needs a lab session, then
starts assembling the report around the screenshots as they arrive. Phase 1 of the AWS brief also
requires presenting the estimate to the lecturer, which is this person's job.

Swap roles afterwards during the video week if everyone wants a full build under their belt.

## Day by day

**Tue 1 Sep** — Stages 01 to 07: region, VPC and six subnets, IGW and NAT and route tables,
security groups, Cloud9, RDS, secret. Start RDS early since it provisions for 5 to 10 minutes while
you carry on. In parallel, the writer starts the architecture diagram.

**Wed 2 Sep** — Stages 08 to 10: phase-2 POC, migration, App Server on RDS. This is the day things
break, so protect it. Remember issue I-01, attach both `App-SG` and `Build-SG` to the stage 10
instance. Capture F-1, F-2 and F-3 as you go, and remember the POC is terminated at stage 10 so its
screenshots are last chance. In parallel, the writer builds the pricing estimate.

**Thu 3 Sep** — Stages 11 to 14: AMI, launch template, ALB, Auto Scaling group. Test a reboot
before creating the AMI. By the end of today the ALB DNS name should serve the application, which
means F-1, LB-1, LB-2 and SC-1 are all captured.

**Fri 4 Sep** — Stage 15 first, then stage 16. Turn on flow logs with the 1-minute aggregation
interval and confirm the status column reads plain Active before moving on, since a silent delivery
failure looks identical to low traffic. Then the load test: budget a full hour, take the baseline
reading before generating any load, and allow 10 to 15 minutes of idle afterwards for scale-in.
Whatever slipped earlier in the week gets caught up today.

**Sat 5 Sep** — Report assembly. The draft in `report-draft.md` already carries the structure, so
this is filling in real values, dropping screenshots against their evidence IDs, and writing the
C7 discussion around the load test numbers. Also the slides from the showcase template.

**Sun 6 Sep** — Final read, convert to PDF, submit.

**Mon 7 to Sun 13 Sep** — Record the video. The stack stays up. Delete everything afterwards.

## Run sheet for the stage 13 to 17 session

Stages 1 to 12 are done. What remains is roughly two and a half hours of console work, which fits
inside one lab session. Two pieces do not need the lab open at all, so do them beforehand and save the
lab time for the console: the AWS Pricing Calculator estimate that stage 17 asks for, and the
architecture diagram in Lucidchart using `architecture-diagram-spec.md`.

**Before touching anything, two minutes of setup.** Check that Build-SG's My IP rules match where you
are sitting today (issue I-05 bites on every network change). Note the new public IPs for App-Server
and Cloud9, since a stopped instance always comes back on a new address. Open the Cloud9 environment
and let it finish initialising, because stage 16 needs it and it takes a minute to wake.

| Stage | Allow | Where the time goes |
|---|---|---|
| 13 Target group and ALB | 20 to 25 min | The load balancer takes 3 to 5 minutes to provision before it will serve anything. |
| 14 Auto Scaling group | 15 to 20 min | The 300-second health check grace period runs before instances report healthy. Do not panic at initial or unhealthy inside that window. |
| 15 Flow logs | under 10 min | Straightforward. |
| 16 Load test | a full hour | Target tracking reacts over several minutes, and scale-in is slower than scale-out. Take the baseline reading before generating any load, and leave 10 to 15 minutes idle at the end for scale-in evidence. |
| 17 Lock down and cost | 20 min in console | Only if the Pricing Calculator work is already done. Remember the I-03 ordering fix: remove the Build-SG source from DB-SG **before** deleting Build-SG. |

**Order matters in two places.** Stage 14 terminates App-Server, so do not start it until the Auto
Scaling instances are serving traffic through the load balancer; if the image turns out to be wrong you
will want App-Server still there. And do not delete Cloud9 at stage 14 or 17 until the load test is
finished, since it is the machine that generates the load.

## If you fall behind

Talk to the group before dropping anything, since all three of you are meant to end up with the
same build. If it comes to it, the order to give things up, worst option last:

1. The second builder's stack. One working stack satisfies every group mark.
2. Scale-in evidence in SC-2. The brief says "where available", so state honestly that you captured
   scale-out but the session ended before scale-in.
3. Depth in the report's discussion sections, keeping the evidence complete.
4. Stage 15, the flow logs, which is the only stage the AWS brief does not require.

Never sacrifice a screenshot to save time. Rebuilding infrastructure to recapture one costs far
more than taking it did.
