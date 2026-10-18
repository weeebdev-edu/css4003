---
theme: seriph
title: "CSS4003 — Lecture 8: AWS Networking"
info: |
  CSS4003 — DevOps 2 · Narxoz University
  Lecture 8 of 15
background: /cover-bg.svg
transition: fade
mdc: true
download: true
---

# CSS4003 — DevOps 2

## Lecture 8: AWS Networking

<div class="pt-8 opacity-70">
Nauruzbayeva Farikha · Lesson 8
</div>

---
layout: default
---

# Recap — Lesson 7 (AWS EC2)

<v-clicks>

- What is an AMI, and why not hard-code its ID? <span v-click class="opacity-60">(the image an instance boots from; IDs differ per region and change with updates, so resolve the latest from SSM)</span>
- When does user data run? <span v-click class="opacity-60">(once, on the first boot, through cloud-init — check `/var/log/cloud-init-output.log` when it fails)</span>
- A security group has no rule for port 22. What happens to SSH? <span v-click class="opacity-60">(it's blocked — inbound is deny-by-default, SGs only have allow rules)</span>
- You stop and start an instance. What changes? <span v-click class="opacity-60">(the auto-assigned public IP; the private IP and the root EBS disk stay)</span>
- Why does user data ask for a token before calling `169.254.169.254`? <span v-click class="opacity-60">(IMDSv2 — a session token protects metadata from SSRF-style requests)</span>

</v-clicks>

---
---

# Today's agenda

<v-clicks>

- [ ] VPCs and CIDR blocks: how big is `10.0.0.0/16`?
- [ ] Subnets, and why each one lives in a single AZ
- [ ] Route tables, the internet gateway, and what makes a subnet "public"
- [ ] Private subnets, NAT gateways and Elastic IPs
- [ ] Security groups vs network ACLs
- [ ] Default VPC vs your own VPC, and a two-tier design
- [ ] S3 gateway endpoints, DNS, and a troubleshooting checklist

</v-clicks>

---
layout: center
class: text-center
---

# The scenario

<div class="text-lg text-left mt-4 max-w-2xl mx-auto">

Your web server from last week works. Now the team adds an app server with
the database credentials on it. Someone launches it into the default VPC,
where every subnet is public, and it gets a public IP. Its security group
allows 22 "from the office" — which turns out to be `0.0.0.0/0`.

</div>

<div v-click class="mt-8 text-xl font-bold">
Today: design a network where only the web tier faces the internet,
and the app tier can't be reached from outside at all.
</div>

---
layout: section
transition: slide-left
---

# Block 1
## VPCs and CIDR

---
---

# A VPC is your private network in a region

<v-clicks>

- **VPC** = Virtual Private Cloud: an isolated network that you define,
  inside one **region**.
- It spans **all the AZs** of that region. Subnets inside it pick an AZ.
- You choose its address range as a **CIDR block**, e.g. `10.0.0.0/16`.
- Traffic between VPCs, or to the internet, only flows where you build a
  path: gateways, routes, peering.
- EC2 instances, RDS databases and load balancers all live **in subnets
  of a VPC**.

</v-clicks>

---
---

# CIDR in one slide

<v-clicks>

- `10.0.0.0/16`: the first **16 bits** are fixed (`10.0`), the rest is
  yours → 2<sup>16</sup> = **65,536** addresses.
- `10.0.1.0/24`: 24 bits fixed → 2<sup>8</sup> = **256** addresses
  (`10.0.1.0` – `10.0.1.255`).
- Each extra bit halves the block: `/25` = 128, `/26` = 64, `/28` = 16.
- A VPC block can be from **`/16`** (largest) to **`/28`** (smallest).

</v-clicks>

<div v-click class="mt-6 p-4 rounded bg-blue-500/10 text-sm">
Rule of thumb: <b>VPC = <code>/16</code>, subnet = <code>/24</code></b>.
That gives up to 256 subnets of 256 addresses each — plenty for this course.
</div>

---
---

# Use private address ranges

| Range (RFC 1918) | Size |
|---|---|
| `10.0.0.0/8` | 16.7 million addresses |
| `172.16.0.0/12` | ~1 million (the default VPC uses `172.31.0.0/16`) |
| `192.168.0.0/16` | 65,536 (common on home routers) |

<v-clicks>

- These ranges are never routed on the public internet, so they're safe
  for internal networks.
- Pick a block that **doesn't overlap** your other VPCs or your office
  network — overlapping ranges can't be connected later.

</v-clicks>

---
---

# AWS keeps 5 addresses in every subnet

For `10.0.1.0/24`:

| Address | Reserved for |
|---|---|
| `10.0.1.0` | Network address |
| `10.0.1.1` | VPC router |
| `10.0.1.2` | Amazon DNS server |
| `10.0.1.3` | Reserved for future use |
| `10.0.1.255` | Broadcast (not supported, still reserved) |

<div v-click class="mt-4 text-sm opacity-70">
So a <code>/24</code> gives <b>251</b> usable addresses, and the smallest
subnet, <code>/28</code>, gives 11.
</div>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
How many usable instance addresses does a <code>10.0.2.0/26</code> subnet
have?
</div>

<div v-click class="mt-8 text-lg opacity-70">
<code>/26</code> = 2<sup>6</sup> = 64 addresses, minus the 5 AWS reserves
= <b>59</b>.
</div>

---
layout: section
transition: slide-left
---

# Block 2
## Subnets, routes and the internet gateway

---
---

# Subnets live in one AZ

<v-clicks>

- A **subnet** is a slice of the VPC's CIDR, e.g. `10.0.1.0/24`.
- Each subnet is in exactly **one Availability Zone**. It can't span two.
- High availability = the same tier in **two subnets in two AZs**.
- Every instance gets its private IP from its subnet's range.

</v-clicks>

```text
VPC 10.0.0.0/16 (us-east-1)
├── 10.0.1.0/24  public   us-east-1a
├── 10.0.2.0/24  private  us-east-1a
├── 10.0.3.0/24  public   us-east-1b
└── 10.0.4.0/24  private  us-east-1b
```

---
---

# Route tables decide where packets go

<v-clicks>

- Every subnet is associated with **one route table**. Subnets without an
  explicit association use the VPC's **main** route table.
- Every route table has a **local** route for the VPC's CIDR. You can't
  delete it — all subnets in a VPC can reach each other.
- More specific routes win: `10.0.0.0/16` beats `0.0.0.0/0`.

</v-clicks>

<div v-click>

| Destination | Target | Meaning |
|---|---|---|
| `10.0.0.0/16` | `local` | Stay inside the VPC |
| `0.0.0.0/0` | `igw-0abc…` | Everything else → internet gateway |

</div>

---
---

# The internet gateway (IGW)

<v-clicks>

- An **IGW** connects a VPC to the internet. One per VPC, managed by AWS,
  scales automatically, no bandwidth limit to plan for.
- Attaching it does nothing on its own. Traffic only uses it when a route
  table sends `0.0.0.0/0` to it.
- For an instance to be reachable from the internet it needs **all three**:
  1. its subnet's route table has `0.0.0.0/0 → igw`
  2. it has a **public IP** (auto-assigned or Elastic IP)
  3. its **security group** allows the port

</v-clicks>

---
---

# What makes a subnet "public"?

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

### Public subnet

- Route table has **`0.0.0.0/0 → igw-…`**
- Usually auto-assigns public IPs
  (`MapPublicIpOnLaunch = true`)
- For: web servers, load balancers,
  bastion hosts, NAT gateways

</div>
<div>

### Private subnet

- Route table has **only `local`** (or
  `0.0.0.0/0 → nat-…`)
- No public IPs
- For: app servers, databases,
  anything that shouldn't be reached
  from outside

</div>
</div>

<div v-click class="mt-6 p-4 rounded bg-blue-500/10 text-sm">
"Public" isn't a checkbox on the subnet. It's <b>the route to the internet
gateway</b>. Change the route table, and the same subnet becomes private.
</div>

---
---

# Building it from the CLI

```bash{1|2-3|4-6|7-9}
VPC=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query Vpc.VpcId --output text)
PUB=$(aws ec2 create-subnet --vpc-id $VPC --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a --query Subnet.SubnetId --output text)
IGW=$(aws ec2 create-internet-gateway \
  --query InternetGateway.InternetGatewayId --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGW --vpc-id $VPC
RT=$(aws ec2 create-route-table --vpc-id $VPC --query RouteTable.RouteTableId --output text)
aws ec2 create-route --route-table-id $RT --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW
aws ec2 associate-route-table --route-table-id $RT --subnet-id $PUB
```

<div v-click class="mt-4 text-sm opacity-70">
Per the Learner Lab Readme, you can create VPCs in Learner Lab — in
<code>us-east-1</code>. That's exactly today's practice.
</div>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
An instance has a public IP and a security group allowing port 80, but
the page never loads. Its subnet's route table has only the
<code>local</code> route. Why?
</div>

<div v-click class="mt-8 text-lg opacity-70">
No <code>0.0.0.0/0 → igw</code> route: the subnet is private, so replies
have no way out to the internet. A public IP alone isn't enough.
</div>

---
layout: section
transition: slide-left
---

# Block 3
## Private subnets, NAT and Elastic IPs

---
---

# Private instances that need to go *out*

<v-clicks>

- An app server in a private subnet sometimes needs outbound internet:
  package updates, calling an external API.
- It must still be **unreachable from the internet**.
- Solution: a **NAT gateway** in a **public** subnet, and in the private
  route table: `0.0.0.0/0 → nat-…`.
- NAT = network address translation: outgoing packets leave with the NAT
  gateway's public IP. Replies come back through it.
- Connections **started from the internet** have nowhere to go — there's
  no route in.

</v-clicks>

---
---

# NAT gateway: what to know

<v-clicks>

- A public NAT gateway sits in a **public subnet** and uses an **Elastic
  IP**.
- It lives in **one AZ**. For high availability: one NAT gateway per AZ.
- It's billed **per hour** while it exists, **plus per GB** processed —
  even when nothing uses it.
- A common surprise on a student bill: a forgotten NAT gateway.
- Today's practice skips NAT: the private instance gets **no** internet
  access at all, which is enough to show the idea and costs nothing extra.

</v-clicks>

<div v-click class="mt-4 text-sm opacity-70">
Check current prices on the AWS VPC pricing page before you create one.
</div>

---
---

# Elastic IP, recap

<v-clicks>

- A **static** public IPv4 address you allocate to your account, then
  associate with an instance or a NAT gateway.
- Survives stop/start; can be moved to another instance in seconds.
- Every public IPv4 address is billed — attached or not. **Release** the
  ones you don't use.

</v-clicks>

```bash
EIP=$(aws ec2 allocate-address --query AllocationId --output text)
aws ec2 associate-address --allocation-id $EIP --instance-id $IID
aws ec2 release-address --allocation-id $EIP     # after disassociating
```

---
layout: section
transition: slide-left
---

# Block 4
## Security groups vs network ACLs

---
---

# Security groups, revisited

<v-clicks>

- Attached to an instance's **network interface**.
- **Stateful**: allow the request in, and the reply goes out automatically.
- **Allow rules only**; everything not allowed is denied.
- A rule's source can be a CIDR **or another security group**:

</v-clicks>

```bash
# app servers accept SSH only from instances in the web security group
aws ec2 authorize-security-group-ingress --group-id $SG_APP \
  --protocol tcp --port 22 --source-group $SG_WEB
```

<div v-click class="mt-4 p-4 rounded bg-blue-500/10 text-sm">
Referencing a group instead of an IP keeps working when instances are
replaced and get new IPs — the rule follows <b>membership</b>, not
addresses.
</div>

---
---

# Network ACLs: a second, subnet-level firewall

<v-clicks>

- Attached to a **subnet**; applies to every instance in it.
- **Stateless**: return traffic must be allowed **explicitly**.
- Has **allow and deny** rules, with numbers. Evaluated **lowest number
  first**; the first match wins.
- The VPC's **default NACL allows everything**. A new custom NACL
  **denies everything** until you add rules.
- Replies go to the client's **ephemeral port** (typically
  1024–65535), so an outbound rule for those ports is needed.

</v-clicks>

---
---

# Security group vs NACL

| | Security group | Network ACL |
|---|---|---|
| Attached to | Instance (network interface) | Subnet |
| State | Stateful | Stateless |
| Rules | Allow only | Allow and deny |
| Evaluation | All rules together | In number order, first match |
| Default | Inbound denied, outbound allowed | Default NACL: all allowed |
| Typical use | Your main firewall | Blocking an IP range for a whole subnet |

<div v-click class="mt-4 text-sm opacity-70">
In this course you'll mostly use security groups and leave the default
NACL alone. Know NACLs exist when a connection fails "for no reason".
</div>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
You add a custom NACL allowing inbound TCP 80. The web page still times
out. The security group is fine. What's missing?
</div>

<div v-click class="mt-8 text-lg opacity-70">
NACLs are stateless: you also need an <b>outbound</b> rule allowing the
replies to the client's ephemeral ports (1024–65535).
</div>

---
layout: section
transition: slide-left
---

# Block 5
## Designing a VPC

---
---

# Default VPC vs your own

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

### Default VPC

- Created by AWS in every region
- `172.31.0.0/16`, one `/20` subnet per AZ
- **All subnets public**, public IPs
  auto-assigned, IGW attached
- Great for quick tests (week 7)

</div>
<div>

### Custom VPC

- You choose the CIDR and subnets
- Public **and** private subnets
- You control every route
- What real projects use

</div>
</div>

<div v-click class="mt-6 p-4 rounded bg-red-500/10 text-sm">
Putting a database or app server in the default VPC means it lands in a
<b>public</b> subnet. One wrong security group rule and it's on the
internet.
</div>

---
---

# A two-tier design

```text
                      Internet
                         │
                  Internet gateway
                         │
┌─ VPC 10.0.0.0/16 ──────┼─────────────────────────────┐
│                        │                             │
│  Public subnet 10.0.1.0/24     route: 0.0.0.0/0 → igw │
│     [ web ]  SG-web: 80 from anywhere, 22 from my IP  │
│        │                                              │
│        │ SSH / app traffic (local route)              │
│        ▼                                              │
│  Private subnet 10.0.2.0/24    route: local only      │
│     [ app ]  SG-app: 22 from SG-web only              │
└───────────────────────────────────────────────────────┘
```

<div v-click class="mt-2 text-sm opacity-70">
This is exactly what you build in today's practice.
</div>

---
---

# Reaching a private instance: the bastion

<v-clicks>

- The app server has **no public IP**. You reach it **through** the web
  server ("bastion" or "jump host").
- Keep your key on your laptop and forward the agent:

</v-clicks>

```bash
ssh-add labsuser.pem                       # load the key into your SSH agent
ssh -A -i labsuser.pem ec2-user@<web-public-ip>
# on the web server:
ssh ec2-user@10.0.2.x                      # private IP of the app server
```

<div v-click class="mt-4 text-sm opacity-70">
In production, <b>SSM Session Manager</b> removes the bastion entirely:
no open SSH port, no keys to copy.
</div>

---
---

# VPC endpoints: reach S3 without the internet

<v-clicks>

- A private instance with no NAT can't reach S3 — S3's public endpoints
  are on the internet.
- A **gateway endpoint** for S3 adds a route to the route table:
  "S3's address ranges → endpoint". Traffic stays on the AWS network.
- Gateway endpoints exist for **S3** and **DynamoDB**, and have no extra
  charge.
- Other services use **interface endpoints** (PrivateLink), which are
  billed.

</v-clicks>

```bash
aws ec2 create-vpc-endpoint --vpc-id $VPC \
  --service-name com.amazonaws.us-east-1.s3 --route-table-ids $PRIV_RT
```

---
---

# DNS inside a VPC

<v-clicks>

- Every VPC has the **Amazon DNS server** at the network base + 2
  (e.g. `10.0.0.2`).
- `enableDnsSupport`: instances can use it (on by default).
- `enableDnsHostnames`: instances with public IPs get public DNS names
  like `ec2-3-80-…compute-1.amazonaws.com`. **On** in the default VPC,
  **off** in a VPC you create — turn it on:

</v-clicks>

```bash
aws ec2 modify-vpc-attribute --vpc-id $VPC --enable-dns-hostnames Value=true
```

---
layout: section
transition: slide-left
---

# Block 6
## Troubleshooting and wrap-up

---
---

# "I can't connect" — check in this order

<v-clicks>

1. **Instance**: running? Status checks 2/2?
2. **Public IP**: does it have one? Is it the *current* one?
3. **Route table**: does its subnet have `0.0.0.0/0 → igw`?
4. **IGW**: attached to *this* VPC?
5. **Security group**: port and source allowed? Your IP changed?
6. **NACL**: default, or a custom one missing a rule (both directions)?
7. **OS**: is the service listening (`ss -tlnp`)? A local firewall?

</v-clicks>

<div v-click class="mt-4 p-4 rounded bg-blue-500/10 text-sm">
<b>Timed out</b> usually means steps 2–6 (packets dropped).
<b>Connection refused</b> means you reached the machine but nothing listens
— step 7.
</div>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
From the web server, <code>ssh</code> to the app server's private IP times
out. Both are in the same VPC. Name the most likely cause.
</div>

<div v-click class="mt-8 text-lg opacity-70">
SG-app doesn't allow port 22 from SG-web (or from the web server's
address). The local route always exists, so routing isn't the problem.
</div>

---
---

# Try it yourself

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

**AWS Academy Learner Lab** (`us-east-1`)

1. VPC `10.0.0.0/16`, a public and a private `/24`
2. IGW + public route table
3. SG-web and SG-app (22 only from SG-web)
4. Web instance in the public subnet, app instance in the private one
5. SSH to web, then hop to app
6. Remove the `0.0.0.0/0` route and watch it break
7. Delete everything, then **End Lab**

</div>
<div>

**Without AWS** (Linux VM or WSL)

Same shape with **network namespaces**:

```bash
sudo ip netns add web
sudo ip netns add app
sudo ip link add br-pub type bridge
sudo ip link add br-priv type bridge
# veth pairs = network interfaces,
# ip route   = route tables,
# iptables   = security groups
```

</div>
</div>

---
---

# By the end of this lesson, you should be able to

<v-clicks>

- [ ] Size a VPC and its subnets with CIDR, and count usable addresses
- [ ] Explain what makes a subnet public or private
- [ ] Build a VPC, subnets, an IGW and a route table from the CLI
- [ ] Say when you need a NAT gateway, and why to delete it after use
- [ ] Compare security groups and NACLs, and use SG-to-SG rules
- [ ] Design a two-tier VPC and reach a private instance through a bastion
- [ ] Debug "can't connect" with a step-by-step checklist

</v-clicks>

---
layout: default
---

# Before next lecture

- [ ] Delete everything from the practice — VPC last — and **End Lab**
- [ ] Make sure no NAT gateway or Elastic IP is left in your account
- [ ] Look at the S3 console: what does a bucket name look like?

<div class="mt-8 text-sm opacity-60">
Next week we add storage and a managed database to this network.
</div>

---
layout: end
---

# Next lecture

AWS Storage & Databases — EBS, S3 and RDS, and where they fit in the VPC
you built today.
