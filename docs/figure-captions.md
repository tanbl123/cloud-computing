# Figure captions

Written as each screenshot is captured. Renumber the figures to match the final report order, but
keep the evidence ID in every caption because the assignment brief asks for it explicitly and it is
the cheapest mark in the assignment.

A caption should say what the image shows **and what it proves**. "Subnets" is a wasted caption.

---

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
restricted to named security groups, and no CIDR range appears as a source, so the database cannot be
reached from the public internet.

**Figure 9.** *(Evidence SEC-1)* Inbound rules for `Build-SG`, the temporary group used to reach
instances directly during construction. Administrative access over SSH was restricted to a single IP
address rather than exposed to `0.0.0.0/0`. This group was deleted once the build was complete.

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
