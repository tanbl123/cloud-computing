# Stage 17 continuation guide (updated)

For whichever teammate is picking up from stage 17 in their own account. This updates the runbook's
Harden and Teardown sections with two things the group has now confirmed the hard way, so you don't lose
time on either. Everything else about stage 17 is unchanged from the original runbook.

This guide does **not** cover the three additional features (middle-tier EC2, cross-region database
migration, SNS scaling notifications) — those are being built and documented separately, on top of a
finished stage 17, and are not required in every teammate's account.

## A. Fix the security-group ordering bug (issue I-03)

The runbook's Harden list has Build-SG deleted before the DB-SG rule that references it is removed. Done
in that order, deleting Build-SG fails with `resource sg-xxxx has a dependent object`, because AWS will
not delete a security group that another group's rule still points at.

Do it in this order instead:

1. EC2 → Security Groups → `DB-SG` → Inbound rules → **Edit inbound rules**.
2. Delete the row whose source is `Build-SG` (the one described as "temporary" or "for build testing").
   Leave the App-SG and Cloud9 SG rows alone.
3. **Save rules.**
4. Screenshot DB-SG's inbound rules now — it should show only two sources left. Keep your original
   three-source screenshot from stage 04/05 too; the two together make a stronger before/after pair for
   the report than either alone.

## B. Check for stray instances before deleting Build-SG (issue I-09, new)

Before you touch Build-SG itself, confirm nothing is still using it. In one account this session,
Build-SG's deletion failed with "1 instance associated, 1 network interface associated" — the culprit was
the stage 08 Phase-2 POC instance (`CapstonePOC`), which the capture plan flags as "**LAST CHANCE
(terminated at stage 10)**" but which had never actually been terminated. It sat running, unnoticed, all
the way through stage 16.

So before deleting Build-SG:

1. EC2 → Instances → filter by **Security group = Build-SG** (or just eyeball your instance list).
2. If your stage 08 Phase-2 POC instance is still there, and you've already confirmed its data was
   migrated into RDS and is being served correctly by your App Server (stage 09–10), terminate it now.
   Nothing depends on it any more.
3. Confirm no other instance in your account uses Build-SG. Your App-ASG instances should all be on
   App-SG, not Build-SG — if one of them shows Build-SG instead, that's a launch template misconfiguration
   worth fixing before continuing.

## C. Delete Build-SG

1. EC2 → Security Groups → select `Build-SG` → **Actions** → **Delete security group** → confirm.
2. It should now succeed with no dependency errors, since both A and B are done.
3. Screenshot the Security Groups list showing the group gone (one fewer group than before) — this is
   the clearest single exhibit that the temporary build-time access has been fully retired, not just
   disabled.

## D. AWS Pricing Calculator — CO-1 and CO-2

1. Go to the AWS Pricing Calculator (calculator.aws) and build a **12-month, us-east-1** estimate covering
   what you actually built: 2× t3.micro EC2 (App-ASG baseline), Application Load Balancer, 1× db.t3.micro
   RDS MySQL (single-AZ, default storage, 7-day backups), 1× NAT Gateway, Secrets Manager (1 secret), S3
   (flow logs, minimal storage).
2. Screenshot the summary (total, 12 months, region) and the full per-service breakdown.
3. Use the calculator's **Export** option to save the estimate as a file too — the report asks for the
   export itself, not just a screenshot.
4. Check your Learner Lab budget page for current spend and note it down.
5. CO-2 (the sizing rationale) is writing, not a capture — the reasons are already in the group's shared
   decision log (D-01 to D-06); transfer them rather than inventing new ones.

## E. Final state checklist

Before you consider stage 17 done in your own account, confirm:

- App-ASG at desired/min capacity 2 (matching the original stage 14 configuration), 2/2 Healthy.
- The stage 08 Phase-2 POC instance and any stage 10 App Server test instances are terminated.
- Cloud9 is left running (or at least not deleted) — it's needed through the video-recording week.
- Build-SG no longer appears in your Security Groups list.
- DB-SG's inbound rules show only App-SG and the Cloud9 SG as sources.
