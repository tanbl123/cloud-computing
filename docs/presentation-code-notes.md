# Code notes for the video presentation

Every command/config step that actually worked, in build order, kept so a speaking script can be
generated from this later. Dead ends (I-10, I-11, the KMS/RDS cross-region attempts) are not included
here — they're documented in the build log instead, and are process findings, not code to present.

## Stage 09 (core build) — original migration

```
mysqldump -h <CapstonePOC private IP or localhost> -u nodeapp -p STUDENTS > data.sql
```
Then imported into RDS from Cloud9 using the RDS endpoint.

## Stage 18 — SNS scaling notifications (bonus, not one of the 3 counted features)

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
- Created lifecycle rule `age-out-old-backups` (entire bucket): noncurrent versions transition to
  Glacier Instant Retrieval after 7 days, permanently deleted after 30 days

Read-only permission checks used to investigate the cross-region block (worth mentioning as the
investigation, not the deliverable):
```
aws iam list-attached-role-policies --role-name LabRole
aws iam list-role-policies --role-name LabRole
aws organizations describe-policy --policy-id p-cnlugp30
aws organizations list-policies-for-target --target-id 401858547100 --filter SERVICE_CONTROL_POLICY
```

## Stage 20 — middle-tier EC2

Console configuration:
- Created `Data-SG`: inbound TCP 3306 from `App-SG` only, outbound all traffic
- Launched `Data-Server` (Ubuntu 24.04 LTS, t3.micro, `asm-App-A`, `Data-SG`, `LabInstanceProfile` for
  Session Manager access, no public IP, no SSH key needed)
- Edited `DB-SG`: removed the rule accepting `App-SG` directly, added one accepting `Data-SG` instead

On Data-Server (via Session Manager, no SSH):
```
sudo apt update && sudo apt install -y socat
sudo nohup socat TCP-LISTEN:3306,fork,reuseaddr TCP:asm-rds.ch9e5pk57w5b.us-east-1.rds.amazonaws.com:3306 &
```
Verify with:
```
ps aux | grep socat
```

Point the app at the middle tier (no code or AMI changes needed, since the app reads its DB host from
Secrets Manager at runtime):
- Secrets Manager → `Mydbsecret` → Edit → changed `host` from the RDS endpoint to Data-Server's private
  IP (`10.0.2.41`)

Follow-up: a Lab session ending stopped Data-Server (a standalone instance, so unlike App-ASG's
instances nothing auto-restarts it). Replaced the ad-hoc `nohup socat` with a persistent systemd
service so it survives future reboots automatically:
```
sudo tee /etc/systemd/system/db-forward.service > /dev/null <<'EOF'
[Unit]
Description=TCP forward to RDS
After=network.target

[Service]
ExecStart=/usr/bin/socat TCP-LISTEN:3306,fork,reuseaddr TCP:asm-rds.ch9e5pk57w5b.us-east-1.rds.amazonaws.com:3306
Restart=always

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now db-forward.service
```

Force the app tier to pick up the change:
- Terminated both running `App-Instance` entries in EC2 → App-ASG launched two replacements
  automatically (the same self-healing mechanism demonstrated in stage 16)

Verification: browsed the ALB URL's `/students` page and confirmed the full list loaded correctly,
proving the request path is now App instance → Data-Server (socat) → RDS.

## Stage 21 — RDS read replica (replaces SNS as the 3rd counted feature)

Console configuration (RDS → `asm-rds` → Actions → Create read replica), all within us-east-1 so the
I-10/I-11 region restriction doesn't apply:
- DB instance identifier: `asm-rds-replica`
- Instance class: db.t3.micro, Single-AZ (not Multi-AZ, to keep cost down)
- Same DB subnet group and `DB-SG` as the primary, not publicly accessible
- Storage autoscaling max lowered from the 1000 GiB default to 50 GiB, Enhanced Monitoring disabled,
  deletion protection off — all deliberate cost/complexity trims

Demonstration: added a record via the app (writes to the primary), then queried the replica's endpoint
directly:
```
mysql -h asm-rds-replica.ch9e5pk57w5b.us-east-1.rds.amazonaws.com -u nodeapp -p STUDENTS -e "SELECT * FROM students;"
```
The new record appeared there too, confirming genuine replication rather than shared storage.

ReplicaLag stayed at ~0 seconds under light traffic, so a burst insert was run directly against the
primary to see if it was even possible to move the needle:
```
for i in $(seq 1 5000); do
  echo "INSERT INTO students (name, address, city, state, email, phone) VALUES ('BurstTest$i', 'Test Address', 'Test City', 'Test State', 'burst$i@example.com', '0000000000');"
done > burst.sql
mysql -h asm-rds.ch9e5pk57w5b.us-east-1.rds.amazonaws.com -u nodeapp -p STUDENTS < burst.sql
```
Even 5,000 rows didn't produce a measurable lag spike — reported honestly as the finding it is, rather
than pushed further: replication easily keeps up at this scale. Cleaned up afterward:
```
mysql -h asm-rds.ch9e5pk57w5b.us-east-1.rds.amazonaws.com -u nodeapp -p STUDENTS -e "DELETE FROM students WHERE email LIKE 'burst%@example.com';"
```

Motivation: stage 16's load test already showed the single RDS instance becoming a real bottleneck
under peak load ("a database with a fixed, small connection ceiling cannot be relieved by adding more
application servers in front of it"). A read replica is the natural next step for read scaling against
that exact limitation — and the ReplicaLag result shows the database layer itself isn't where that
bottleneck comes from, the compute tier is.

## Stage 22 — Pricing Calculator (CO-1, CO-2)

No code — AWS Pricing Calculator (calculator.aws), us-east-1, 12-month view. Eight line items priced to
match the built architecture exactly:

- App-ASG baseline: 2x t3.micro, Shared tenancy, On-Demand
- Data-Server (middle tier): 1x t3.micro, Shared tenancy, On-Demand
- asm-rds (primary): db.t3.micro, Single-AZ, 20 GiB gp3, no Proxy/Database Insights/Extended Support
- asm-rds-replica: same spec as the primary, added as its own RDS line item
- asm-alb: 1 Application Load Balancer
- asm-NAT: 1 NAT Gateway, 10 GB/month processed
- Mydbsecret: 1 secret in Secrets Manager
- asm-db-backups bucket: S3 Standard, 1 GB

Total: $102.42/month, $1,229.04 over 12 months. Exported as a file alongside the summary screenshot.
