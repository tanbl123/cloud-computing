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

**Drop stage 15, the VPC flow logs.** It is the only part of the runbook that the AWS project brief
does not require. Do it in the video week if there is time and you want the extra C5 material.

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

**Fri 4 Sep** — Stage 16, the load test. Budget a full hour and take the baseline reading before
generating any load. Scale-in needs 10 to 15 minutes of idle afterwards. Whatever slipped earlier
in the week gets caught up today.

**Sat 5 Sep** — Report assembly. The draft in `report-draft.md` already carries the structure, so
this is filling in real values, dropping screenshots against their evidence IDs, and writing the
C7 discussion around the load test numbers. Also the slides from the showcase template.

**Sun 6 Sep** — Final read, convert to PDF, submit.

**Mon 7 to Sun 13 Sep** — Record the video. The stack stays up. Delete everything afterwards.

## If you fall behind

The order to sacrifice things, worst option last:

1. Stage 15, the flow logs. Not required at all.
2. Scale-in evidence in SC-2. The brief says "where available", so state honestly that you captured
   scale-out but the session ended before scale-in.
3. The second builder's stack. One working stack is enough for every group mark.
4. Depth in the report's discussion sections, keeping the evidence complete.

Never sacrifice a screenshot to save time. Rebuilding infrastructure to recapture one costs far
more than taking it did.
