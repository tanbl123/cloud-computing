# Figure captions

Written as each screenshot is captured. Renumber the figures to match the final report order, but
keep the evidence ID in every caption because the assignment brief asks for it explicitly and it is
the cheapest mark in the assignment.

A caption should say what the image shows **and what it proves**. "Subnets" is a wasted caption.

---

## Architecture diagram

**Figure A.** *(Evidence HA-1)* Target architecture of the student-records web application in
`us-east-1`. The VPC `asm-VPC` is divided into three tiers, each repeated in `us-east-1a` and
`us-east-1b`: public subnets carrying the Application Load Balancer, private application subnets
carrying the Auto Scaling group, and private database subnets carrying Amazon RDS. The database
subnets have no route to the internet, so the database is unreachable from outside the VPC by
routing rather than only by security group rules. Application instances reach AWS service endpoints
outbound through the NAT gateway, and retrieve their database credentials from AWS Secrets Manager
at runtime using the IAM role attached to their instance profile, so no password is stored in the
application code or in the machine image.

Place this as the first figure in section 4, before the design rationale.

## Stage 02 — VPC and subnets

**Figure 1.** *(Evidence HA-1)* The six subnets of `asm-VPC`, showing three tiers distributed across
two Availability Zones. The public subnets host the load balancer, the application subnets host the
web instances and the database subnets host Amazon RDS, with each tier repeated in `us-east-1a` and
`us-east-1b` so that no tier depends on a single zone. Auto-assign public IPv4 is enabled on the two
public subnets only.

**Figure 2.** *(Evidence HA-1)* Configuration of `asm-VPC`, showing the `10.0.0.0/16` address range
and DNS hostnames enabled. DNS hostnames is required for instances to resolve the Amazon RDS
endpoint by name.

## Stage 03 — Route tables

**Figure 3.** *(Evidence SEC-1)* Route table `asm-RT-Public`. The `0.0.0.0/0` route to internet
gateway `asm-IGW` is what makes `asm-Public-A` and `asm-Public-B` public subnets, and the subnet
associations confirm that only those two subnets follow this table.

**Figure 4.** *(Evidence SEC-1)* Route table `asm-RT-App`. Outbound traffic from the application
subnets leaves through NAT gateway `asm-NAT` rather than the internet gateway. This gives the web
instances access to AWS service endpoints such as Secrets Manager while permitting no inbound
connection from the internet.

**Figure 5.** *(Evidence SEC-1)* Route table `asm-RT-DB`. The table contains a single route,
`10.0.0.0/16` to local, and no default route of any kind. The database subnets therefore have no
path to the internet by routing, independently of any security group rule.

## Stage 04 — Security groups

**Figure 6.** *(Evidence SEC-1)* Inbound rules for `ALB-SG`. The load balancer is the only component
that accepts traffic from the public internet, and only on HTTP port 80.

**Figure 7.** *(Evidence SEC-1)* Inbound rules for `App-SG`. The source is the `ALB-SG` security
group rather than an IP range, so only the load balancer can reach the application instances. Because
the rule names a group rather than an address, it remains correct as the Auto Scaling group changes
size.

**Figure 8.** *(Evidence SEC-1)* Inbound rules for `DB-SG`. Access to MySQL on port 3306 is
restricted to three named security groups: the application tier, the temporary build group, and the
Cloud9 administration host. No CIDR range appears as a source, so the database cannot be reached from
the public internet.

**Figure 9.** *(Evidence SEC-1)* Inbound rules for `Build-SG`, the temporary group used to reach
instances directly during construction. Browser and SSH access were restricted to a single
administrative IP address rather than exposed to `0.0.0.0/0`, and MySQL access was granted only to the
Cloud9 host so that the database could be exported during migration. This group was deleted once the
build was complete.

## Stage 05 — Cloud9

**Figure 10.** *(Evidence HP-1)* The AWS Cloud9 environment `asm-Cloud9`, a `t3.micro` instance placed
inside `asm-VPC`. Cloud9 provided the only command line available in the lab environment, and was used
to create the Secrets Manager secret, perform the database migration, and generate load during
performance testing.

## Stage 06 — Amazon RDS

**Figure 11.** *(Evidence HA-2)* The DB subnet group `asm-db-subnet-group`, spanning `asm-DB-A` in
`us-east-1a` and `asm-DB-B` in `us-east-1b`. Amazon RDS requires a subnet group covering at least two
Availability Zones before a database can be created, which is why the second database subnet exists
even though this deployment runs a single-AZ instance.

**Figure 12.** *(Evidence SEC-1)* Connectivity configuration of the `asm-rds` database instance.
Publicly accessible is set to No and the only attached security group is `DB-SG`, so the database has
no public IP address and accepts connections only from the sources named in that group. The instance
sits in the private subnets of `asm-db-subnet-group`, which have no route to the internet.

**Figure 13.** *(Evidence HA-2)* Backup configuration of `asm-rds`. Automated backups are enabled with
a seven-day retention period, and the first automated snapshot has already been taken, so the
configuration is demonstrably working rather than merely set. This supports point-in-time recovery to
any moment within the retention window.

## Stage 07 — Secrets Manager

**Figure 14.** *(Evidence SEC-2)* The secret `Mydbsecret` in AWS Secrets Manager, holding the database
username, password, host endpoint and database name. The `describe-secret` call returns the secret's
identity and metadata but never its value, which is itself a demonstration of the access model: the
credential can be referenced and audited without being read. The application retrieves the value at
runtime through the IAM role attached to its instance profile.

## Stage 08 — Phase-2 proof of concept

**Figure 15.** *(Evidence F-1)* The student records application running on the single EC2 instance
`CapstonePOC` in `asm-Public-A`, reached directly by its public IPv4 address. This is the Phase 2
proof of concept required by the project brief: the application and its database on one virtual
machine, before the tiers were separated. The connection is plain HTTP, which the assignment scope
permits for a proof of concept.

**Figure 16.** *(Evidence F-2, read)* The student records list served from `CapstonePOC`, showing six
records retrieved from the MySQL database running on the same instance.

**Figure 17.** *(Evidence F-2, create)* A new record for Chong Mei Ling submitted through the
application form, and the list reloaded afterwards showing the record persisted with a city and state
of Melaka.

**Figure 18.** *(Evidence F-2, update)* The same record edited to change the city to Seremban and the
state to Negeri Sembilan, with the list reloaded afterwards showing the amended values. Reloading
confirms the change was written to the database rather than only reflected in the form.

**Figure 19.** *(Evidence F-2, delete)* The confirmation prompt for removing Chong Mei Ling, and the
list reloaded afterwards showing the record gone and the six original records intact.

Together Figures 16 to 19 demonstrate all four record operations required by the functional criterion,
each verified by reloading the page after the write rather than by the form's own response.

**Figure 20.** *(Evidence F-3, before migration)* The state of both databases before the migration. The
upper query, run against the phase-2 instance at `10.0.0.184`, returns the six student records held in
its local MySQL. The lower query, run against the Amazon RDS endpoint, returns no tables at all,
confirming the managed database was still empty at this point. The record identifiers run from 1 to 6
with no 7, showing that the deletion demonstrated in Figure 19 removed the row from the database
rather than only from the page.

## Stage 09 — Migration

**Figure 21.** *(Evidence F-3, after migration)* The migration and its verification. The first command
exports the `STUDENTS` database from the phase-2 instance at `10.0.0.184`, the second imports it into
Amazon RDS, and the third queries the RDS endpoint directly and returns all six records with every
column intact. Read against Figure 20, where the same RDS query returned no tables at all, this
confirms the data was moved into the managed database rather than having been created there.

## Stage 10 - The App Server on Amazon RDS

**Figure 22.** *(Evidence SEC-2)* The App Server instance summary. The IAM role field shows `LabRole`,
attached at launch through the `LabInstanceProfile` instance profile. This role is the only mechanism
by which the instance is able to obtain database credentials, as no credentials of any kind are stored
on the instance itself.

**Figure 23.** *(Evidence SEC-2)* The user data script with which the App Server was built, retrieved
directly from the instance attribute using the AWS CLI. The script installs the MySQL client only, with
no database server, and the sole environment variable it sets is `APP_PORT`. No database hostname,
username or password appears anywhere in it, so the application has neither a local database nor any
stored credential to fall back on and must obtain its credentials from AWS Secrets Manager at run time.
The closing lines register the application start command in `/etc/rc.local`, which is what allows the
instance to serve traffic after a restart and, in turn, allows the machine image created in Stage 11 to
launch unattended.

> How to take Figure 23. The console route, Actions then Instance settings then Edit user data, refuses
> to open the editor while the instance is running. Retrieve the attribute from Cloud9 instead:
> `aws ec2 describe-instance-attribute --instance-id i-0636b1e60f111d310 --attribute userData --query "UserData.Value" --output text | base64 -d`
> This prints the script that the instance was actually built with, straight from the EC2 API, which is
> stronger evidence than a copy of the file. If you take it from the console after stopping the
> instance instead, change "retrieved from the instance attribute using the AWS CLI" to "viewed through
> the console" in the caption.

**Figure 24.** *(Evidence F-2)* The application served by the App Server instance at `54.227.135.18`,
displaying the six student records migrated to Amazon RDS in Stage 9. Because this instance runs no
database engine and holds no credentials in its configuration, the records shown can only have been
read from the managed database using the username and password retrieved from AWS Secrets Manager at
run time.

**Figure 25.** *(Evidence F-2)* A seventh student record created through the App Server. Read together
with Figure 24, this demonstrates that the instance both reads from and writes to Amazon RDS, and that
the application layer holds no state of its own.

**Figure 26.** *(Evidence F-2)* The same seven records returned by a query issued from the Cloud9
environment directly against the Amazon RDS endpoint. As Cloud9 is a separate instance querying the
database rather than the application, this confirms that the record created in Figure 25 was persisted
to the managed database and not to storage local to the App Server. The separation of the application
tier from the data tier is what allows any web server to be replaced without loss of data, which is the
property the Auto Scaling group depends upon in Stage 14. The gap between identifiers 6 and 9 is
expected: identifiers 7 and 8 were consumed by records created and then deleted during the CRUD
demonstration in Stage 8, and MySQL does not reissue an auto-increment value once used. That the new
record continued the sequence from the phase-2 instance rather than restarting confirms that the
migration carried the table's auto-increment state and not only its rows.

**Figure 27.** *(Evidence F-2)* The application responding correctly after the App Server was rebooted.
This verifies that the instance configuration survives a restart, which is a precondition for creating
the machine image in Stage 11, since every instance launched by the Auto Scaling group is created from
that image and must start unattended.

## Stage 11 - The machine image

**Figure 28.** *(Evidence SC-1)* The `App-Server-AMI-v1` machine image, `ami-0d28ed0cb650c6cf4`, in the
Available state. It was created from the App Server instance only after that instance had been verified
against Amazon RDS and shown to restart unattended. Every instance the Auto Scaling group launches in
Stage 14 is created from this image, which is what allows capacity to be added or replaced with no
manual configuration. The Source AMI ID shown, `ami-0f8a61b66d1accaee`, records that the image derives
from Ubuntu Server 24.04 LTS.

## Stage 12 - The launch template

**Figure 29.** *(Evidence SC-1, C3)* The `App-LT-v1` launch template, `lt-06c91acc74c3b008f`, specifying
the verified machine image, the `t3.micro` instance type and the `App-SG` security group. The
Availability Zone field is empty by design: because no zone or subnet is pinned in the template, the
Auto Scaling group is free to distribute instances across both availability zones, which is what makes
the deployment tolerant of the loss of either one.

**Figure 30.** *(Evidence SEC-2, C3)* The advanced details of the same launch template, showing the
`LabInstanceProfile` IAM instance profile that every launched instance receives. No user data is
specified, since the application and its start-up entry are already contained in the machine image, and
no credentials appear anywhere in the template.

## Stage 13 - Target group and load balancer

**Figure 31.** *(Evidence LB-1, C2)* The `App-TG` target group, configured for HTTP traffic on port 80
within `asm-VPC` and associated with the `asm-ALB` load balancer. Instances are registered to this group
automatically by the Auto Scaling group rather than by hand, so that capacity added or removed by
scaling is reflected in the load balancer without manual intervention.

**Figure 32.** *(Evidence LB-1, C2)* The health check configuration of `App-TG`. The load balancer
requests the application root path over HTTP every thirty seconds and expects a 200 response. A target
is considered healthy after five consecutive successes and is removed from rotation after two
consecutive failures, at which point the Auto Scaling group replaces it. This is the mechanism that
makes the deployment self-healing rather than merely redundant.

**Figure 33.** *(Evidence LB-1, C2)* The `asm-ALB` Application Load Balancer in the Active state. It is
internet-facing and spans two availability zones, with a node in `asm-Public-A` in `us-east-1a` and
another in `asm-Public-B` in `us-east-1b`, so the loss of an entire zone does not remove the entry point
to the application. Its DNS name is the single public address for the system and remains stable when the
instances behind it are replaced.

**Figure 34.** *(Evidence LB-1, C2)* The listener on `asm-ALB`, accepting HTTP requests on port 80 and
forwarding all of them to the `App-TG` target group. Because routing is defined against a target group
rather than against named instances, the load balancer requires no reconfiguration when the Auto Scaling
group adds or removes capacity. Target group stickiness is off, which is appropriate because the
application holds no session state of its own and any instance can serve any request.

## Stage 14 - Auto Scaling group

**Figure 35.** *(Evidence LB-2, C2, C3)* The `App-TG` target group's registered targets, showing both
Auto Scaling instances, `i-00c957897434a961` in `us-east-1a` and `i-0af80d429a31ff2a2` in `us-east-1b`,
reporting Healthy. The differing Availability Zone values confirm that the Auto Scaling group
distributed its instances across both zones as configured, and the Healthy status on each confirms the
group's health checks and the application itself are both functioning correctly.

**Figure 36.** *(Evidence F-1, C1)* The completed application served through `asm-ALB`, showing all
seven student records. The address in the bar is the load balancer's DNS name, not an instance IP; the
browser has no way of knowing, and does not need to know, which of the two instances behind it answered
the request. This is the point at which the architecture becomes genuinely highly available: either
instance can fail and the application remains reachable at the same address.

**Figure 37.** *(Evidence SC-1, C3)* The `App-ASG` Auto Scaling group's details, showing a desired
capacity of 2 with scaling limits of 2 to 4, at desired capacity with both instances healthy across two
availability zones. The launch template panel confirms every instance the group creates inherits the
verified `App-LT-v1` configuration, including the machine image, security group and instance profile,
so scaling requires no manual configuration of new instances.

**Figure 38.** *(Evidence SC-1, C3)* The target tracking scaling policy attached to `App-ASG`, configured
to maintain average CPU utilization at 50 percent by adding or removing instances as required, with a
300-second instance warmup before new instances contribute to the metric. Scale-in remains active (the
"Scale in" field reports the disable-scale-in override as off, not that scaling in is disabled), so the
group is expected to grow under the load test in Stage 16 and then shrink back toward its desired
capacity once load subsides.

---

## Note on the DB-SG screenshot

`DB-SG` changes three times, so decide which state you are documenting:

| After | Sources on port 3306 |
|---|---|
| Stage 04 | App-SG, Build-SG |
| Stage 05 | App-SG, Build-SG, Cloud9 SG |
| Stage 17 | App-SG, Cloud9 SG |

The **stage 17 version is the strongest exhibit**, because Build-SG is gone and what remains is the
least-privilege end state. Take one now as insurance, then retake it at stage 17 and use that in the
report, adjusting Figure 8's caption to describe the final configuration.
