# BMIT3273 Cloud Computing (202605)

Group assignment: **Building a Highly Available, Scalable Web Application**, built in the AWS
Academy Lab Project: Cloud Web Application Builder.

Submission is a PDF report plus a live demonstration. The coursework is worth 50% of the module,
split 70 group marks across criteria C1 to C7 and 30 individual marks per student.

## What is in here

| File | What it is | When you use it |
|---|---|---|
| [`docs/capture-plan.md`](docs/capture-plan.md) | Build and capture plan. Pairs every runbook stage with the screenshots it must produce. | **Open this while you build.** |
| [`docs/build-log.html`](docs/build-log.html) | Group build log: agreed values, stage progress, decision log, issue log, evidence tracker. | Before you build, and after each stage. |
| [`docs/report-draft.md`](docs/report-draft.md) | Report content draft, sections 3 to 10, in markdown. | When writing the report. |
| `docs/BMIT3273-Report-Content-Draft.docx` | The same draft as a Word file with placeholders highlighted. | When writing the report. |
| [`docs/stage-17-continuation-guide.md`](docs/stage-17-continuation-guide.md) | Updated stage 17 steps: the correct Build-SG deletion order, a stray-instance check the original runbook misses, and the Pricing Calculator steps. | Picking up stage 17 in your own account. |

The 17-stage build runbook itself lives outside this repo, alongside the assignment brief and the
lecture decks in the shared Google Drive folder.

## Before you touch the AWS console

**Read section 2 of the build log first.** It holds the agreed resource names, CIDRs and the values
that are hardcoded in the lab scripts. All three of us build the same architecture in separate lab
accounts, so if one person uses different values the screenshots stop matching and the report
becomes impossible to write consistently. Several of those values are not ours to choose:
`nodeapp`, `student12`, `STUDENTS`, `Mydbsecret` and port 80 are fixed by the lab scripts and
changing them breaks the migration.

**Read issue I-01 in section 5 of the build log.** Stage 10 of the runbook launches the App Server
with `Build-SG` only, but `DB-SG` does not accept traffic from `Build-SG`. The instance will boot,
fetch its secret successfully, and then fail every database operation, which looks like an
application bug and is not. Attach both `App-SG` and `Build-SG` to that instance.

## How we work

Build and capture at the same time. The capture plan gives every screenshot a filename, states what
has to be visible in the frame for it to count, and names the report section it belongs to. Save
files under exactly those names and the report assembles itself later.

**Six pieces of evidence are destroyed by a later stage.** They are marked LAST CHANCE in the
capture plan. The ones that catch people are the phase-2 POC instance, terminated at stage 10, the
`Build-SG` rules, deleted at stage 17, and the standalone App Server, terminated at stage 14.
Missing any of them means rebuilding infrastructure purely to take a photograph.

Screenshots go in an `evidence/` folder in your own working copy, not in this repo, since they are
large and each of us has our own set. Record non-screenshot values, such as the RDS endpoint, the
ALB DNS name and your load test numbers, in an `evidence/values.md` as you go.

After finishing a stage, update your column in section 3 of the build log with the date. Log any
problem in section 5 even after you fix it, because the other two will probably hit the same thing.

## Budget

The NAT gateway, the ALB and the RDS instance together cost roughly US$2.50 a day and keep billing
whether or not anyone is logged in. A Learner Lab budget does not survive that for the whole
semester, and each of us is running a separate copy.

Follow stage 17 between sessions: set the Auto Scaling group desired and minimum to 0, and delete
the NAT gateway and release its Elastic IP if the budget is tight. Recreating it takes three
minutes. Do not delete anything the day before the demonstration.

## Status

One account has completed all 17 stages, including load testing (stage 16, run three times — see
the build log) and security lock-down (stage 17). See `docs/stage-17-continuation-guide.md` for the
other two accounts picking up stage 17. That account is now building all three additional features required of the group (worth 5 marks) as
stages 18 to 20 — SNS scaling notifications, then cross-region database migration, then a
middle-tier EC2 between the app and database (done last since it's the riskiest, changing the live
traffic path) — followed by the Pricing Calculator estimate (CO-1, CO-2) as stage 21, done last so
it prices in what the additional features add.

Everything in `docs/` that is written by Claude carries bracketed or highlighted placeholders where
a real value from the lab account is required. Do not submit anything with those still in place,
and do not fill them in by guesswork, since the marker cross-checks the text against the
screenshots.
