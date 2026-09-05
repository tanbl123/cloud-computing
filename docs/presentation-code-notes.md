# Code notes for the video presentation

Every command/config step that actually worked, in build order, kept so a speaking script can be
generated from this later. Dead ends (I-10, I-11, the KMS/RDS cross-region attempts) are not included
here — they're documented in the build log instead, and are process findings, not code to present.

## Stage 09 (core build) — original migration

```
mysqldump -h <CapstonePOC private IP or localhost> -u nodeapp -p STUDENTS > data.sql
```
Then imported into RDS from Cloud9 using the RDS endpoint.

## Stage 18 — SNS scaling notifications

No code — console configuration only:
- Created SNS topic `asm-scaling-notifications` (Standard type)
- Created an email subscription, confirmed via the link in the confirmation email
- On `App-ASG` → Activity notifications → Create notification → selected the topic → checked all six
  event types (Launch, Terminate, Launch error, Terminate error, Replace root volume, Fail to replace
  root volume)
- Triggered real events by manually editing App-ASG's desired capacity 2 → 3 → 2

## Stage 19 — backup versioning and lifecycle (pivoted from cross-region DB migration)

Export the database to a file:
```
mysqldump -h asm-rds.ch9e5pk57w5b.us-east-1.rds.amazonaws.com -u nodeapp -p STUDENTS > cross-region-backup.sql
```

Upload it to the versioned S3 bucket under a fixed key, so re-uploads create new versions instead of new
objects:
```
aws s3 cp cross-region-backup.sql s3://asm-db-backups-401858547100/latest-backup.sql
```

Repeated after changing data in the app (added a student record), to produce a genuine second version.

Console configuration:
- Created bucket `asm-db-backups-401858547100` with Bucket Versioning **Enabled**
- (Next) added a lifecycle rule on noncurrent versions: transition to a cheaper storage class after ~7
  days, expire after ~30 days

Read-only permission checks used to investigate the cross-region block (worth mentioning as the
investigation, not the deliverable):
```
aws iam list-attached-role-policies --role-name LabRole
aws iam list-role-policies --role-name LabRole
aws organizations describe-policy --policy-id p-cnlugp30
aws organizations list-policies-for-target --target-id 401858547100 --filter SERVICE_CONTROL_POLICY
```

## Stage 20 — middle-tier EC2 (to be filled in as we build it)
