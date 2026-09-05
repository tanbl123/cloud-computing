# Video presentation speaking script

Written to be read aloud (adapt the wording to sound natural in your own voice) while you screen-share
the console and Cloud9. Each section names what to have on screen before you start talking. Numbers,
resource names and commands are pulled directly from the build log, values.md and
`presentation-code-notes.md` — nothing here is invented.

Re-check the Learner Lab budget page immediately before recording and update the number in the Cost
section below with whatever it actually reads at that moment.

---

## 1. Introduction (30–45 seconds)

*On screen: title slide or the architecture diagram.*

"Hi, I'm [name], and this is our group's submission for BMIT3273 Cloud Computing — Building a Highly
Available, Scalable Web Application, using the AWS Academy Cloud Web Application Builder lab.

Our architecture is a three-tier student records web application: a load-balanced, auto-scaling web
tier across two availability zones, a MySQL database on RDS, and a set of additional features we added
on top of the required build — backup versioning, a middle-tier proxy layer, and an RDS read replica.
I'll walk through the architecture, demonstrate it live, and finish with the cost estimate."

---

## 2. Architecture overview (1–2 minutes)

*On screen: the architecture diagram.*

"The VPC, `asm-VPC`, spans two availability zones with six subnets: two public, two for the application
tier, and two for the database tier. Only the two public subnets have a route to the internet gateway,
`asm-IGW`. The application subnets reach the internet only outbound, through a single NAT gateway,
`asm-NAT`, in one of the public subnets — that's for package updates and calls to Secrets Manager, not
inbound traffic. The database subnets have no default route at all — not even outbound — so even a
misconfigured security group can't give the database a path to the internet. That's defence in depth
beyond just the security group rules.

Traffic comes in through an internet-facing Application Load Balancer, `asm-ALB`, listening on port 80
across both public subnets, forwarding to a target group backed by an Auto Scaling group, `App-ASG`,
running a minimum of 2 and maximum of 4 t3.micro instances, one per availability zone at minimum. Scaling
is target-tracking on 50% average CPU.

Each application instance is built from a single AMI and reads its database connection details from AWS
Secrets Manager at runtime — not from environment variables, not hardcoded. That single design decision
is what let us insert an entire new tier into the request path later without touching a single line of
application code, which I'll show in a minute.

The database itself, `asm-rds`, is a single MySQL db.t3.micro instance, and we added a read replica,
`asm-rds-replica`, as one of our additional features.

Security groups are chained, not CIDR-based: the load balancer accepts the internet, the app tier only
accepts the load balancer, and the database tier only accepts specific application-layer security
groups — never a raw IP range. That's one of our strongest defence-in-depth arguments."

---

## 3. Live demo: the application (1 minute)

*On screen: browser at the ALB DNS name.*

"Here's the live application at our ALB's DNS name,
`asm-ALB-743581282.us-east-1.elb.amazonaws.com` — this URL is stable across lab sessions. This is our
student records system. I can view the list, add a new record, edit one, and delete one — full CRUD,
served by whichever of the two-to-four application instances the load balancer happens to route me to."

*(Demonstrate one add/edit/delete cycle live if time allows.)*

---

## 4. How credential resolution actually works (30–45 seconds)

*On screen: Secrets Manager console, `Mydbsecret`.*

"One thing worth explaining, because it's the mechanism behind two of our additional features: this app
copies its database settings from environment variables first, but then asynchronously fetches the
`Mydbsecret` secret from Secrets Manager and overwrites those settings. The secret always wins. So
whether an instance talks to a local database or to RDS is decided entirely by whether it has an IAM
role that can read this secret — nothing else. That's the whole decoupling story, and it's also exactly
why we could redirect all database traffic through a brand-new middle-tier instance later without
touching the application AMI at all — we just changed the `host` field in this secret."

---

## 5. High availability and self-healing (30 seconds)

*On screen: Auto Scaling group activity history, or EC2 instance list across both AZs.*

"For high availability: application instances run across both availability zones behind the load
balancer, so losing one AZ doesn't take the app down. During our load test, we also saw the Auto Scaling
group replace an instance that the underlying EC2 layer had stopped outside its own control — proof the
self-healing isn't limited to CPU-triggered scaling, it also recovers from infrastructure-level failures."

---

## 6. Load test results (30–45 seconds)

*On screen: CloudWatch CPU graph and/or load test terminal output from stage 16.*

"We load tested with up to 1000 requests per second and 500 concurrent connections using the `loadtest`
tool from Cloud9. The application and load balancer tiers scaled cleanly, but the database — a single
db.t3.micro RDS instance — became the real bottleneck under peak load: a database with a fixed, small
connection ceiling can't be relieved just by adding more application servers in front of it. That
finding is exactly what motivated our third additional feature, the read replica, which I'll get to
shortly."

---

## 7. Additional feature 1 — backup versioning and lifecycle management (1–1.5 minutes)

*On screen: S3 bucket `asm-db-backups-401858547100`, Versions tab, then the lifecycle rule.*

"Our first additional feature is S3-based backup versioning and lifecycle management. We originally
planned a cross-region database migration for this feature, but ran into a hard restriction in this
Academy Lab: RDS actions like `CopyDBSnapshot`, and even read-only calls like `DescribeDBInstances`, are
explicitly denied by an organisation-level service control policy the moment you target any region other
than us-east-1. We confirmed this with real attempts — a snapshot copy, a fresh RDS instance in Ohio, and
even a brand-new S3 bucket in Ohio — all blocked. We even tried to inspect *why*, using
`aws organizations describe-policy` and `aws iam get-policy` on our own role's policies, and those calls
are denied too. This Lab doesn't just block cross-region deployment, it blocks you from seeing the policy
that blocks it.

So we pivoted to a same-region alternative that's still a genuine, demonstrable feature: we exported the
database with

```
mysqldump -h asm-rds.ch9e5pk57w5b.us-east-1.rds.amazonaws.com -u nodeapp -p STUDENTS > cross-region-backup.sql
```

and uploaded it to a versioned S3 bucket under a fixed key:

```
aws s3 cp cross-region-backup.sql s3://asm-db-backups-401858547100/latest-backup.sql
```

Uploading to the same key twice, after changing the underlying data in between, produces a genuine new
object version rather than a new object — you can see both versions here in the S3 console. On top of
that, we configured a lifecycle rule, `age-out-old-backups`, that transitions noncurrent versions to
Glacier Instant Retrieval after 7 days and permanently deletes them after 30 — an automated backup
retention policy, not just a one-off backup."

---

## 8. Additional feature 2 — middle-tier EC2 (1–1.5 minutes)

*On screen: Data-SG rules, Data-Server instance, DB-SG's updated rule, the socat process.*

"Our second additional feature inserts a new proxy tier between the application and the database. We
created a new security group, `Data-SG`, that only accepts MySQL traffic on port 3306 from the
application security group, then launched a new instance, `Data-Server`, using that security group and
the Lab's instance profile — which gives it Systems Manager Session Manager access, so we never needed
SSH or an open port 22. We then updated `DB-SG` itself to only accept traffic from `Data-SG`, removing
its direct route from the application tier.

On the Data-Server instance, over Session Manager, we installed and ran `socat` as a simple TCP
forwarder:

```
sudo apt update && sudo apt install -y socat
sudo nohup socat TCP-LISTEN:3306,fork,reuseaddr TCP:asm-rds.ch9e5pk57w5b.us-east-1.rds.amazonaws.com:3306 &
```

Then — and this is the part that ties back to how credential resolution works — we didn't touch the
application AMI or any code at all. We just edited the `Mydbsecret` secret's `host` field to point at
Data-Server's private IP, `10.0.2.41`, instead of the RDS endpoint directly. We terminated the running
application instances so the Auto Scaling group launched replacements that picked up the new secret
value, and confirmed the `/students` page still loads correctly end-to-end — now routed through
App instance → Data-Server → RDS, instead of straight to RDS."

---

## 9. Additional feature 3 — RDS read replica (1.5–2 minutes)

*On screen: `asm-rds-replica` details page, then the ReplicaLag CloudWatch graph.*

"Our third additional feature is a read replica for `asm-rds`, and this one directly follows from the
load test finding I mentioned earlier — the database was our real bottleneck, not the compute tier. We
created `asm-rds-replica`, same region, db.t3.micro, single-AZ to keep cost down, with storage
autoscaling capped at 50 GiB rather than the 1000 GiB default, Enhanced Monitoring disabled, and deletion
protection off — deliberate cost and complexity trims, since this is a demonstration, not a production
system.

To prove genuine replication rather than shared storage, we added a record through the live application
— which writes to the primary — and then queried the replica's own endpoint directly:

```
mysql -h asm-rds-replica.ch9e5pk57w5b.us-east-1.rds.amazonaws.com -u nodeapp -p STUDENTS -e "SELECT * FROM students;"
```

and the new record was there. Then we checked `ReplicaLag` in CloudWatch, which stayed at essentially
zero seconds throughout. We pushed it deliberately with a burst insert of 5,000 rows straight to the
primary:

```
for i in $(seq 1 5000); do
  echo "INSERT INTO students (name, address, city, state, email, phone) VALUES ('BurstTest$i', 'Test Address', 'Test City', 'Test State', 'burst$i@example.com', '0000000000');"
done > burst.sql
mysql -h asm-rds.ch9e5pk57w5b.us-east-1.rds.amazonaws.com -u nodeapp -p STUDENTS < burst.sql
```

Even that didn't move the lag needle. We're reporting that honestly rather than manufacturing a bigger
number: at this scale, MySQL asynchronous replication on a small instance keeps up easily. It's actually
an informative result in its own right — it confirms our stage 16 finding that the compute tier is where
this architecture's real limit is, not the database's replication layer."

---

## 10. Cost estimate (1 minute)

*On screen: the AWS Pricing Calculator summary page (Figure 75).*

"For the cost estimate, we used the AWS Pricing Calculator for a 12-month, us-east-1 projection covering
the base architecture plus all three additional features. Eight line items: the App Auto Scaling group
baseline at two t3.micro instances, the Data-Server middle-tier instance, the RDS primary and the read
replica — both db.t3.micro — the Application Load Balancer, the NAT Gateway, Secrets Manager, and the S3
backup bucket. That comes to $102.42 a month, or $1,229.04 across 12 months.

A few sizing decisions worth explaining: we used one NAT gateway rather than one per availability zone —
two would remove a cross-AZ dependency, but our outbound traffic here is only package updates and
Secrets Manager calls, so it wasn't worth doubling that line item. The Auto Scaling group's minimum of 2
is an availability floor, not a performance target — one instance can't survive a failure. The maximum
of 4 keeps a load test or a traffic spike bounded, and this Learner Lab also caps concurrent instances at
9 regardless. And RDS is single-AZ rather than Multi-AZ, partly because Multi-AZ is blocked in this
Learner Lab, and partly because the brief itself assumes single-AZ — in a production deployment, Multi-AZ
would be the first upgrade we'd make.

As of recording, our actual Learner Lab spend is [state the live number from the budget page here] out
of the $100 lab budget."

---

## 11. Lab constraints and closing (30–45 seconds)

*On screen: back to the architecture diagram, or a closing slide.*

"A few Academy Lab restrictions shaped this build: VPC Flow Logs had to go to S3 instead of CloudWatch,
because this Lab's role isn't trusted by the Flow Logs service; Multi-AZ RDS and cross-region deployment
are both blocked outright; and a Web Application Firewall isn't available in this Lab, though we've kept
it on our diagram as a designed control, since Shield Standard is automatic on the load balancer
regardless. Each of these is a case where we made the best available production-equivalent decision
within the Lab's constraints, and we've documented all of them.

That's our build — a highly available, auto-scaling three-tier web application with three additional
features that genuinely extend it: backup versioning and lifecycle management, a middle-tier proxy layer,
and a read replica for database read scaling. Thank you."

---

## Notes for delivery

- Total script reads at roughly 9–12 minutes at a natural pace — trim section 2 or 6 first if you need
  to shorten, since sections 7–10 (the additional features and cost) are the ones the lecturer explicitly
  wants explained.
- Swap in your own name and adjust "we" / "I" depending on whether teammates appear in the recording.
- If a teammate is presenting from your build (since only one full build is needed for the demo), have
  them read this as their own script — everything here describes what's actually in this account, not a
  hypothetical.
