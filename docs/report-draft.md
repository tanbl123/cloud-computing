# BMIT3273 Cloud Computing (202605): Report Content Draft

> **How to use this file.** This is drafting material for the Google Doc skeleton, written to
> match the section numbering already in the document. Everything in `[SQUARE BRACKETS]` must be
> replaced with a real value taken from your own AWS Academy lab account. Every figure reference
> (F-1, LB-1, SC-2, …) must point at a screenshot you actually captured. Text that describes a
> setting you did not configure should be deleted, not reworded. An unsupported claim costs more marks than an omission.
>
> **Assumed design.** The draft assumes the architecture below. If your build differs, change the
> numbers here first, then the prose will still hold:
>
> | Component | Assumed value |
> |---|---|
> | Region | us-east-1 |
> | VPC CIDR | 10.0.0.0/16 |
> | Public subnets (ALB) | 10.0.0.0/24 (us-east-1a), 10.0.1.0/24 (us-east-1b) |
> | Private app subnets | 10.0.2.0/24 (us-east-1a), 10.0.3.0/24 (us-east-1b) |
> | Private DB subnets | 10.0.4.0/24 (us-east-1a), 10.0.5.0/24 (us-east-1b) |
> | Web tier | EC2 t3.micro, Ubuntu, behind an Auto Scaling group (min 2 / desired 2 / max 4) |
> | Load balancer | Internet-facing Application Load Balancer, HTTP:80 |
> | Database | Amazon RDS for MySQL, db.t3.micro, single AZ, private subnet group |
> | Credentials | AWS Secrets Manager, read at runtime via LabInstanceProfile / LabRole |
> | Admin tooling | AWS Cloud9 (t3.micro) for AWS CLI work and load testing |

---

## 3. Introduction and system overview

Example University's admissions department maintains a web application that its staff use to look
up and maintain student records. The application performs adequately for most of the academic
year, but during the admissions intake the volume of enquiries rises sharply and users report that
pages load slowly or fail to load at all. The underlying cause is structural rather than
functional: the application and its database run on a single virtual machine, so there is one
server absorbing all traffic, one copy of the data, and no mechanism for adding capacity when
demand rises. A single fault or a single busy afternoon is enough to make the service unavailable.

This report documents a proof of concept that our group built in the AWS Academy Lab Project –
Cloud Web Application Builder environment to address that problem. The deliverable is a working
student-records application that supports the four record operations the admissions staff depend on: viewing, adding, modifying, and deleting student records. It is deployed on an architecture that distributes traffic across multiple servers, adds and removes servers automatically as load
changes, keeps the database off the public internet, and stores its database credentials outside
the application code.

The finished system is organised into three tiers inside a single Amazon Virtual Private Cloud
(VPC) spanning two Availability Zones. An internet-facing Application Load Balancer in the public
subnets is the only component that accepts traffic from the internet. It forwards requests to a
pool of Amazon EC2 instances managed by an EC2 Auto Scaling group in the private application
subnets; these instances run the application code and render the user interface. Those instances
in turn connect to an Amazon RDS for MySQL instance in a separate set of private subnets, which
holds the student-records data. Database credentials are held in AWS Secrets Manager and retrieved
at runtime through an IAM role attached to the instance profile, so no password appears in the
application source. Amazon CloudWatch provides the metrics that drive scaling decisions and the
evidence used in our performance testing.

The application is reachable at `[ALB DNS NAME]` (see F-1). In line with the scope set out in the
assignment brief, the proof of concept is deployed in one Region, uses HTTP rather than HTTPS with
no custom domain, and is publicly accessible without user authentication. Section 8 revisits these
limitations and identifies what a production deployment would need to add.

---

## 4. Architecture diagram and design rationale

*(Insert the architecture diagram here and label it **HA-1**. It must show: the VPC boundary; both
Availability Zones; public and private subnets; the ALB; the Auto Scaling group and its instances;
the RDS instance; the internet gateway; the NAT gateway if you used one; and Secrets Manager. Draw
it with the official AWS architecture icons.)*

### 4.1 Network design

The whole solution sits inside one VPC using the CIDR block `[10.0.0.0/16]`, which gives ample
address space for the subnets required now and leaves room for growth. The VPC is divided into
three subnet groups, each repeated in `[us-east-1a]` and `[us-east-1b]`.

The **public subnets** hold only the load balancer `[and the NAT gateway]`. They route to the
internet gateway, which is what makes the application reachable from outside AWS. Placing nothing
else in these subnets keeps the internet-facing surface of the system as small as possible.

The **private application subnets** hold the EC2 instances that run the application. These subnets
have no route to the internet gateway, so the instances cannot be reached directly from the
internet no matter what their security group allows; the only path to them is through the load balancer. `[Outbound access for package installation and Secrets Manager calls is provided by a NAT
gateway in the public subnet.]`

The **private database subnets** hold the RDS instance through its DB subnet group. Separating
these from the application subnets means the database has its own route table and its own security
boundary, and it makes the security group rules easy to reason about: the database accepts traffic
from the application security group and from nothing else.

The address plan is set out below. Every subnet is repeated in both Availability Zones so that no
tier depends on a single zone.

| Subnet | CIDR | AZ | Type | Route table | What it holds |
|---|---|---|---|---|---|
| `asm-Public-A` | 10.0.0.0/24 | us-east-1a | Public | `asm-RT-Public` | ALB node, NAT gateway |
| `asm-Public-B` | 10.0.1.0/24 | us-east-1b | Public | `asm-RT-Public` | ALB node |
| `asm-App-A` | 10.0.2.0/24 | us-east-1a | Private | `asm-RT-App` | Auto Scaling web instances |
| `asm-App-B` | 10.0.3.0/24 | us-east-1b | Private | `asm-RT-App` | Auto Scaling web instances |
| `asm-DB-A` | 10.0.4.0/24 | us-east-1a | Private | `asm-RT-DB` | RDS subnet group |
| `asm-DB-B` | 10.0.5.0/24 | us-east-1b | Private | `asm-RT-DB` | RDS subnet group |

What makes a subnet public or private is not the name but the route table attached to it, so those
are given in full:

| Route table | Destination | Target | Associated subnets |
|---|---|---|---|
| `asm-RT-Public` | 10.0.0.0/16 | local | `asm-Public-A`, `asm-Public-B` |
| | 0.0.0.0/0 | `asm-IGW` (internet gateway) | |
| `asm-RT-App` | 10.0.0.0/16 | local | `asm-App-A`, `asm-App-B` |
| | 0.0.0.0/0 | `asm-NAT` (NAT gateway) | |
| `asm-RT-DB` | 10.0.0.0/16 | local | `asm-DB-A`, `asm-DB-B` |

The database route table is the one worth reading carefully, because **its significance is what is
absent**. It carries only the automatic local route, so there is no path from the database subnets
to the internet in either direction. This is a stronger guarantee than a security group rule,
because a security group can be widened by a later edit whereas a missing route means the packets
have nowhere to go. Section 6.5 returns to this as the primary C5 evidence.

*(These tables state the design; figures SEC-1 and HA-1 prove it was built. Keep both, since the
route table console view is hard to read at screenshot size and the missing default route on
`asm-RT-DB` is easier to see stated than photographed.)*

> **Adjust this paragraph to what you actually did.** If the lab budget or service restrictions
> prevented you from running a NAT gateway, you most likely placed the application instances in the
> public subnets instead. That is a legitimate choice in a cost-constrained proof of concept, and
> it is better to explain it honestly than to claim a private tier you did not build. The
> replacement rationale is: *"NAT gateway charges accrue hourly regardless of traffic and would
> have consumed a disproportionate share of the lab budget, so the application instances were
> placed in the public subnets with their security group restricted to accept traffic only from the
> load balancer's security group. This preserves the practical effect of the private tier (no direct inbound access from the internet) at no additional cost, while the database remains in
> genuinely private subnets."* Then make sure your diagram matches.

Spreading each tier across two Availability Zones is the single most important decision in the
design. An Availability Zone is a physically separate set of facilities with independent power and
networking (Amazon Web Services, n.d.-i), so a fault confined to one zone leaves the resources in
the other zone running. Because the load balancer only sends requests to targets that pass its
health check, the loss of a zone removes those targets from rotation and traffic continues through
the survivors without manual intervention.

### 4.2 Compute tier

The application runs on `[t3.micro]` EC2 instances launched from a launch template, under the
control of an Auto Scaling group. Two decisions are worth explaining.

First, the instances are **stateless and interchangeable**. All persistent data lives in RDS, and
the instances hold no session state that would be lost when one is terminated. This is what makes
automatic scaling safe: the Auto Scaling group can add or remove an instance at any moment without
the user noticing, because any instance can serve any request. Building the AMI with the
application code and its dependencies already installed means a replacement instance reaches a
healthy state quickly rather than spending several minutes installing packages at boot.

Second, the **launch template is the single definition of what a server is**. Rather than
configuring instances individually, the launch template fixes the AMI, instance type, security
group, and instance profile, so every instance the group launches is identical to the one we
tested. This removes configuration drift as a source of failure.

### 4.3 Database tier

We chose Amazon RDS for MySQL over continuing to run MySQL on an EC2 instance. The managed service
handles patching, backups, and recovery, and it separates the database's lifecycle from the application servers'. That separation is precisely what allows the application tier to be replaced freely by Auto Scaling. Keeping the database on one of the application instances would have made every
scaling event a risk to the data.

The instance is `[db.t3.micro]` in a single Availability Zone, as the assignment scope permits.
This is the one deliberate availability compromise in the design, and Section 6.4 discusses it
directly. Automated backups are retained for `[7]` days, which provides point-in-time recovery
within that window.

### 4.4 Credential handling

The application does not contain a database password. At startup it retrieves the credentials from
AWS Secrets Manager using the permissions granted by `[LabRole]`, which is attached to the
instances through `[LabInstanceProfile]`. This matters for two reasons beyond the obvious one.
Because the credential is never written into the AMI, an AMI that leaks does not leak the database.
And because the permission is granted to a role rather than to a stored key, there is no long-lived
access key on the instance to be stolen (Amazon Web Services, n.d.-g).

### 4.5 How the design meets each requirement

| Requirement | Mechanism |
|---|---|
| Functional | Application tier serving CRUD operations against RDS MySQL |
| Load balanced | Internet-facing ALB distributing across targets in two AZs |
| Scalable | EC2 Auto Scaling group with a target tracking policy |
| Highly available | Web tier in two AZs; ALB health checks; managed database with backups |
| Secure | Database in private subnets; tiered security groups; credentials in Secrets Manager |
| Cost optimised | Burstable instance types; Auto Scaling minimum held at the smallest resilient size |
| High performing | Horizontal scale-out under load, validated by load testing |

---

## 5. AWS Pricing Calculator estimate and cost discussion

*(Insert the AWS Pricing Calculator summary here and label it **CO-1**. It must be a 12-month
estimate for us-east-1. Export the estimate and put the full breakdown in the appendix.)*

### 5.1 Estimate summary

| Service | Configuration | Monthly `[USD]` | 12-month `[USD]` |
|---|---|---|---|
| Amazon EC2 | `[2 × t3.micro, on-demand, 730 hrs]` | `[  ]` | `[  ]` |
| Application Load Balancer | `[1 ALB, LCU estimate]` | `[  ]` | `[  ]` |
| Amazon RDS for MySQL | `[db.t3.micro, single AZ, 20 GB gp3]` | `[  ]` | `[  ]` |
| `[NAT gateway]` | `[1 gateway + data processing]` | `[  ]` | `[  ]` |
| AWS Secrets Manager | `[1 secret]` | `[  ]` | `[  ]` |
| Data transfer | `[estimate]` | `[  ]` | `[  ]` |
| **Total** | | `[  ]` | `[  ]` |

### 5.2 Sizing rationale (evidence CO-2)

The sizing follows from the workload's shape rather than from a target price. The admissions
application has a low steady baseline and a pronounced seasonal peak, which is the profile
burstable instance types are designed for: `[t3.micro]` instances accrue CPU credits during quiet
periods and spend them during short bursts, so we are not paying continuously for capacity that is
only needed occasionally.

The Auto Scaling **minimum of `[2]`** is set by availability rather than by load. One instance
would be sufficient for baseline traffic, but it would also mean that losing a single instance, or a single Availability Zone, takes the application offline until a replacement finishes booting.
Two instances in two zones is the smallest configuration that survives that failure, so the second
instance is the cost of the availability requirement, not spare capacity.

The **maximum of `[4]`** is a budget control as much as a capacity ceiling. It bounds what a
traffic spike, a runaway script, or a load test can cost. We set it from our load test results:
`[N]` instances were sufficient to hold response times steady at our peak test rate, and the
maximum allows headroom above that.

The database is sized at `[db.t3.micro]` because the working set is small and the query pattern is
simple record lookups and single-row writes. Storage is provisioned at `[20 GB]`, the minimum that
comfortably holds the dataset, because RDS storage is billed on what is provisioned rather than on
what is used.

### 5.3 Cost trade-offs we accepted

Two decisions traded resilience for budget, and both should be read as scope choices rather than
oversights. We ran the database in a **single Availability Zone**; a Multi-AZ deployment would
roughly double the database cost for a standby instance that serves no traffic, and the assignment
scope permits single-AZ. `[We also ran a single NAT gateway rather than one per Availability Zone,
which leaves cross-AZ outbound traffic dependent on one zone; for a proof of concept whose outbound
traffic is limited to package updates and Secrets Manager calls, we judged the saving worthwhile.]`

The largest cost lever we did **not** pull is committed-use pricing. Reserved Instances or a Savings
Plan would materially reduce the EC2 and RDS lines over twelve months, but they commit to a
one- or three-year term, which does not suit a proof of concept whose architecture is still
changing. We note it here as the first thing to revisit before a production deployment.

---

## 6. Criterion Evidence

### 6.1 C1 Functional

The application is reachable from the public internet at `[ALB DNS NAME]` (Figure F-1) and supports
all four record operations required by the admissions workflow.

| Operation | What was tested | Result | Evidence |
|---|---|---|---|
| View | Loaded the student list and opened an individual record | `[Pass]` | `[Fig. F-2a]` |
| Add | Created a record for `[test student]` | `[Pass]` | `[Fig. F-2b]` |
| Modify | Edited `[field]` on an existing record and reloaded to confirm persistence | `[Pass]` | `[Fig. F-2c]` |
| Delete | Removed a record and confirmed it no longer appears in the list | `[Pass]` | `[Fig. F-2d]` |

Each write operation was verified by reloading the page after the action, so the screenshots show
the state persisted in the database rather than a client-side response.

Data migrated from the original single-instance MySQL database is present and queryable in RDS.
Figure F-3 shows `[the students table with N rows queried directly against the RDS endpoint]`,
confirming that the migration carried the records across rather than starting from an empty schema.

Under normal use, page operations completed in approximately `[X]` seconds `[measured from the
browser developer tools Network tab]`, with no operation exhibiting a delay perceptible to the
user.

> **Fill this section from real observations.** Take the response time from your browser's Network
> tab or from the ALB `TargetResponseTime` metric in CloudWatch, and state which one you used.

### 6.2 C2 Load Balanced

An internet-facing Application Load Balancer named `[ALB NAME]` is the entry point to the
application (Figure LB-1). It is deployed across `[us-east-1a]` and `[us-east-1b]`, which means the
service has a node in each zone and can direct traffic to either.

The listener configuration is deliberately simple: a single HTTP listener on port 80 with a default
action forwarding all requests to the target group `[TG NAME]`. Because the proof of concept serves
one application with no path-based routing requirement, no additional listener rules were needed.

The target group registers instances on port `[80]` with a health check on `[path /]`, considering
a target healthy after `[2]` consecutive successful checks at `[30]`-second intervals and unhealthy
after `[2]` failures. The health check is the mechanism that connects load balancing to
availability: the load balancer routes only to targets currently passing it, so an instance that fails, whether through an application error or the loss of its Availability Zone, is removed from rotation automatically, without a person intervening (Amazon Web Services, n.d.-c).

Figure LB-2 shows `[N]` registered targets, all reporting *healthy*, distributed across both
Availability Zones. Distribution was confirmed during the load test described in Section 6.7:
`[state how you confirmed it; for example, the CloudWatch NetworkIn or CPUUtilization metric rose
on both instances simultaneously, or the per-instance request count in the target group metrics
showed comparable volumes]`.

> **Do not skip that last sentence.** The rubric asks specifically for evidence of *request
> distribution*, not just of a load balancer existing. A per-instance CloudWatch graph over the
> load test window is the cleanest proof.

### 6.3 C3 Scalable

The application tier scales horizontally through an EC2 Auto Scaling group (Figure SC-1) built on
the launch template `[TEMPLATE NAME]`, which fixes the AMI `[AMI ID]`, instance type `[t3.micro]`,
security group, and the `[LabInstanceProfile]` instance profile.

The group is configured with a minimum of `[2]`, a desired capacity of `[2]`, and a maximum of
`[4]`, spread across `[us-east-1a]` and `[us-east-1b]`. The rationale for these values is given in
Section 5.2: the minimum is set by the requirement to survive the loss of one instance or one zone,
and the maximum bounds the cost of a demand spike.

Scaling is driven by a **target tracking policy** on `[average CPU utilisation]` with a target of
`[50]` percent. Target tracking was chosen over step scaling because it requires only a statement
of the desired steady state rather than a set of thresholds and adjustments; the service then
computes the capacity changes needed to hold the metric near the target and scales in
conservatively to avoid oscillation (Amazon Web Services, n.d.-b). For a workload with a smooth
demand curve, this produces stable behaviour with far less configuration to get wrong.

During the load test, average CPU utilisation across the group rose to `[X]` percent, exceeding the
target and triggering a scale-out from `[2]` to `[N]` instances. The new instances passed their
health checks and began receiving traffic approximately `[X]` minutes after the alarm entered the
ALARM state. Figure SC-2 shows `[the Activity history of the Auto Scaling group with the scale-out
events]`. `[After load generation stopped, utilisation fell below the target and the group scaled
back in to N instances after the cooldown period.]`

> **On scale-in evidence.** Scale-in is deliberately slower than scale-out, so you may need to leave
> the environment idle for 10–15 minutes after the load test to capture it. If you could not
> observe it within the session, say so explicitly rather than implying you did. The rubric says *"where available"*.

### 6.4 C4 Highly Available

Availability in this design rests on removing single points of failure from the request path and
on making failures recoverable without human action.

**Multi-AZ web tier.** The Auto Scaling group launches instances in subnets in two Availability
Zones and the load balancer has nodes in both, so the loss of one zone leaves a complete serving
path intact in the other (Figure LB-2, Figure HA-1). Because the Auto Scaling group works to
maintain its desired capacity, an instance that fails its health check is terminated and replaced
automatically.

**Tiered network separation.** The architecture diagram (HA-1) shows public subnets carrying only
the load balancer, private subnets carrying the application instances, and separate private subnets
carrying the database. This limits the blast radius of a compromise or misconfiguration at any one
tier.

**Decoupled data tier.** Moving the database out of the application instance and into RDS is what
makes the application tier disposable. Any instance can be terminated at any time by a scaling event, an instance failure, or a deployment, and no data is lost, because no instance owns any data.

**Recovery readiness.** Automated backups are enabled on the RDS instance with a retention period
of `[7]` days and a backup window of `[window]`, which supports point-in-time recovery to any
moment within the retention window (Amazon Web Services, n.d.-f). Figure HA-2 shows this
configuration.

**Acknowledged limitation.** The database runs in a single Availability Zone. If that zone becomes
unavailable, the application tier survives but has no database to serve from, so the database is
the remaining single point of failure in the design. The assignment scope permits this, and Section
5.3 explains the cost reasoning, but it should be stated plainly: a production deployment would
enable Multi-AZ, which maintains a synchronous standby in a second zone and fails over
automatically. We treat this as the highest-priority enhancement rather than a completed control.

> **Say this out loud in the demo too.** Naming your own architecture's weak point, with the fix and
> the reason you deferred it, reads as engineering judgement. Claiming high availability you did not
> build is the thing that gets caught in questioning.

### 6.5 C5 Secure

Security controls are applied at the network boundary, at the credential layer, and at the
administrative access path.

**Layered security groups.** Three security groups form a chain in which each tier accepts traffic
only from the tier in front of it (Figure SEC-1):

| Security group | Inbound | Source | Purpose |
|---|---|---|---|
| `[alb-sg]` | TCP 80 | `0.0.0.0/0` | The application is intended to be publicly reachable |
| `[app-sg]` | TCP `[80]` | `[alb-sg]` | Only the load balancer may reach the application |
| `[db-sg]` | TCP 3306 | `[app-sg]` | Only application instances may reach MySQL |

The important detail is that the sources are **security group references, not CIDR ranges**. A rule
naming `[app-sg]` as its source applies automatically to every instance the Auto Scaling group
launches, and to no other resource. The rule therefore stays correct as the fleet changes size, and it
cannot be widened accidentally by an IP address being reused.

**Private database placement.** The RDS instance is deployed in a DB subnet group made up of
private subnets with no route to the internet gateway, and it is `[not publicly accessible]`.
Reachability was verified by `[state how; for example, attempting a connection to the RDS endpoint
from outside the VPC and observing that it fails, while the same connection from an application
instance succeeds]`. The network placement and the security group are independent controls, and the
database is protected by both.

**Externalised credentials.** The database username and password are stored in AWS Secrets Manager
as `[SECRET NAME]` and retrieved by the application at runtime through the IAM role attached to the
instance profile (Figure SEC-2). No credential is present in the application source, in the launch
template's user data, or in the AMI.

**Restricted administrative access.** Administrative work was performed from `[the AWS Cloud9
environment / an SSH connection from CIDR X]` rather than by exposing SSH to the internet.
`[Describe your actual arrangement; for example, port 22 is not open to 0.0.0.0/0 on any security
group, and administrative sessions originate from the Cloud9 environment inside the VPC.]`

**Known gaps within scope.** Traffic between the browser and the load balancer is unencrypted HTTP,
and the application requires no user authentication. Both are permitted by the assignment scope but
would be unacceptable for a system holding real student records; Section 8 sets out what would be
required.

### 6.6 C6 Cost Optimized

*(The estimate and its rationale are in Section 5. Cross-reference rather than repeating: cite
**CO-1** for the calculator export and **CO-2** for the sizing rationale, and summarise in two or
three sentences here.)*

Cost was treated as a design constraint from Phase 1 rather than as something assessed after
building. The three decisions that most affect the twelve-month figure are the use of burstable
instance types matched to a bursty workload, an Auto Scaling minimum set to the smallest
configuration that still survives an instance or zone failure, and a single-AZ database. Each is
justified in Section 5, and the maximum capacity of `[4]` acts as a ceiling on what unexpected
demand can cost.

### 6.7 C7 High Performing

**Method (evidence HP-1).** Load was generated with `[the loadtest tool, run from the AWS Cloud9
environment using Script-2 from the Cloud9 scripts file]`, targeting the load balancer DNS name so
that requests entered the system by the same path a real user would take. `[State the command and
its parameters: concurrency, request rate, and duration.]` Metrics were collected from
`[the load test tool's own output and the CloudWatch metrics for the ALB and the Auto Scaling
group]`. Each test ran for `[X]` minutes, with `[a settling period between runs to allow the group
to return to its baseline capacity]`.

**Results (evidence HP-2).**

| Condition | Concurrency / rate | Mean response `[ms]` | `[p95]` response `[ms]` | Errors | Instances |
|---|---|---|---|---|---|
| Normal | `[  ]` | `[  ]` | `[  ]` | `[  ]` | `[2]` |
| Variable | `[  ]` | `[  ]` | `[  ]` | `[  ]` | `[  ]` |
| Peak | `[  ]` | `[  ]` | `[  ]` | `[  ]` | `[  ]` |

**Discussion.** `[Write this once you have your numbers. The pattern to describe, if your results
show it: response times stay flat at normal load; as the request rate rises past what two instances
can serve, response times increase and CPU utilisation crosses the scaling target; the Auto Scaling
group adds instances; once those instances pass their health checks and start receiving traffic,
response times fall back toward the baseline. The delay between the load increasing and capacity arriving is the interesting part, so quantify it. It is a real property of the system, and knowing it is what distinguishes analysis from a screenshot.]`

`[If you recorded errors or timeouts, report them and explain when they occurred, typically in the
window between demand rising and new capacity becoming healthy. Reporting a clean run you did not
have is both a plagiarism-adjacent problem and easy to expose in questioning.]`

---

## 7. Demonstration and student contribution section

*(Complete this from your group's actual division of work, and label it **DEMO-1**.)*

| No. | Student name | Student ID | Main contributions | Demo role | Evidence ref. |
|---|---|---|---|---|---|
| 1 | `[  ]` | `[  ]` | `[  ]` | `[  ]` | `[  ]` |
| 2 | `[  ]` | `[  ]` | `[  ]` | `[  ]` | `[  ]` |
| 3 | `[  ]` | `[  ]` | `[  ]` | `[  ]` | `[  ]` |

Each member should also attach their own evidence for D2 (tutorial and practical submissions) and
D3 (AWS guided lab and challenge lab completion) in the appendices, since those are marked
individually.

> **Note on D1.** Marks go to explaining and defending your own work, so each member should be able
> to answer questions about a component they personally built. Split the demo along the same lines
> as the build.

---

## 8. Conclusion

The proof of concept meets the requirements set out in the brief. The student-records application
is publicly reachable, supports the full set of record operations against a migrated database, and
runs on an architecture in which no web-tier component is a single point of failure. Traffic is
distributed by an Application Load Balancer across instances in two Availability Zones, capacity
adjusts automatically to demand through EC2 Auto Scaling, the database is isolated in private
subnets and reachable only from the application tier, and database credentials are held in AWS
Secrets Manager rather than in code. Load testing confirmed that the system `[summarise the result
in one clause]`.

The exercise also clarified a distinction that is easy to miss when reading about cloud
architecture rather than building it. Availability and scalability are not features that a service
provides; they are properties that follow from how components are arranged. The Auto Scaling group
can replace a failed instance only because the instances hold no state, and they hold no state only
because the database was moved out to RDS first. The decoupling in Phase 3 is what made the
resilience in Phase 4 possible.

Three enhancements would be required before a system of this kind could hold real student records.
The most significant is **Multi-AZ deployment of the database**, which would remove the last single
point of failure in the architecture at the cost of a standby instance. The second is **transport
encryption and authentication**: a certificate from AWS Certificate Manager with an HTTPS listener
would protect data in transit, and access control would restrict the application to authenticated staff rather than the whole internet, which is a necessity for personal data regardless of the technical architecture. The third is **network-level restriction** of the application to the university's own
network ranges, so that the service is not exposed publicly at all. Beyond these, `[a read replica
would relieve the database of read traffic if the record-lookup load grew, and committed-use pricing
would reduce the running cost once the architecture stabilised]`.

---

## 9. References

> Verify every URL before submitting and check your faculty's required APA edition, since the 7th edition does not require retrieval dates for stable content. Add any AWS Academy lab guides or
> textbooks you actually drew on, and delete any entry below that you did not use.

Amazon Web Services. (n.d.-a). *AWS Well-Architected Framework*.
https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html

Amazon Web Services. (n.d.-b). *Amazon EC2 Auto Scaling User Guide*.
https://docs.aws.amazon.com/autoscaling/ec2/userguide/what-is-amazon-ec2-auto-scaling.html

Amazon Web Services. (n.d.-c). *Application Load Balancers*.
https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html

Amazon Web Services. (n.d.-d). *Amazon EC2 User Guide*.
https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html

Amazon Web Services. (n.d.-e). *Amazon Virtual Private Cloud User Guide*.
https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html

Amazon Web Services. (n.d.-f). *Amazon RDS User Guide*.
https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Welcome.html

Amazon Web Services. (n.d.-g). *AWS Secrets Manager User Guide*.
https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html

Amazon Web Services. (n.d.-h). *Amazon CloudWatch User Guide*.
https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html

Amazon Web Services. (n.d.-i). *Regions and Zones*.
https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html

Amazon Web Services. (n.d.-j). *AWS Pricing Calculator User Guide*.
https://docs.aws.amazon.com/pricing-calculator/latest/userguide/what-is-pricing-calculator.html

---

## 10. Appendices: evidence checklist

Label each artefact with its evidence ID in the caption, as the brief requires.

| ID | Artefact | Status |
|---|---|---|
| F-1 | Public URL / ALB DNS landing page | `[ ]` |
| F-2 | CRUD proof (view, add, edit, delete) | `[ ]` |
| F-3 | Migrated data present in RDS | `[ ]` |
| LB-1 | ALB, listener, and target group configuration | `[ ]` |
| LB-2 | Healthy targets across two AZs | `[ ]` |
| SC-1 | Launch template, ASG limits, scaling policy | `[ ]` |
| SC-2 | Scale-out (and scale-in) event proof | `[ ]` |
| HA-1 | Architecture diagram | `[ ]` |
| HA-2 | RDS backup / recovery settings | `[ ]` |
| SEC-1 | Security group rules for all three tiers | `[ ]` |
| SEC-2 | Secrets Manager secret and application integration | `[ ]` |
| CO-1 | 12-month Pricing Calculator export (us-east-1) | `[ ]` |
| CO-2 | Sizing rationale | `[ ]` |
| HP-1 | Load test method | `[ ]` |
| HP-2 | Performance results | `[ ]` |
| DEMO-1 | Student contribution log | `[ ]` |
