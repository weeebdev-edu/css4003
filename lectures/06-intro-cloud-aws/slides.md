---
theme: seriph
title: "CSS4003 — Lecture 6: Introduction to Cloud & AWS"
info: |
  CSS4003 — DevOps 2 · Narxoz University
  Lecture 6 of 15
background: /cover-bg.svg
transition: fade
mdc: true
download: true
---

# CSS4003 — DevOps 2

## Lecture 6: Introduction to Cloud & AWS

<div class="pt-8 opacity-70">
Adil Akhmetov · Lesson 6
</div>

---
layout: default
---

# Recap — Lesson 5 (Advanced Git & GitHub)

<v-clicks>

- Why create a hotfix branch from `main`, not from `develop`? <span v-click class="opacity-60">(`main` is what's in production — `develop` carries unfinished work you don't want to ship with the fix)</span>
- A commit pushed to a shared branch broke things. `git revert` or `git reset`? <span v-click class="opacity-60">(`revert` — it adds a new inverse commit instead of rewriting history other people already pulled)</span>
- Three local commits vanished after `git reset --hard HEAD~3`. Where do you look? <span v-click class="opacity-60">(`git reflog` — it records where `HEAD` has been, including the "lost" commits)</span>

</v-clicks>

---
---

# Today's agenda

<v-clicks>

- [ ] What "cloud" actually means — and what it doesn't
- [ ] Service models (IaaS / PaaS / SaaS) and deployment models
- [ ] The shared responsibility model
- [ ] AWS global infrastructure: regions, Availability Zones, edge
- [ ] A tour of the core services we'll use in weeks 7–9
- [ ] Keeping your account safe and your bill at zero
- [ ] Console vs CLI → straight into today's practice

</v-clicks>

---
layout: center
class: text-center
---

# The scenario

<div class="text-lg text-left mt-4 max-w-2xl mx-auto">

Your team's app from the Git weeks is ready to go live. The old plan: buy a
server, wait two weeks for delivery, rack it, install Linux, and hope it's
big enough. A classmate says "just put it on AWS". Another one opened an
AWS account last year, left a server running, and got a bill.

</div>

<div v-click class="mt-8 text-xl font-bold">
Today is about both halves of that story: why the cloud is the default
answer, and how to use it without getting burned.
</div>

---
layout: section
transition: slide-left
---

# Block 1
## What cloud computing is

---
---

# Cloud = someone else's data center, rented by API

<v-clicks>

- You don't buy hardware. You **request** compute, storage, or a database
  — from a web console, a CLI, or code — and get it in minutes.
- You pay for what you use, while you use it. Turn it off, the meter stops
  (mostly — we'll get to the exceptions).
- The provider owns the buildings, power, cooling, networking, and
  physical servers.

</v-clicks>

<div v-click class="mt-8 p-4 rounded bg-blue-500/10 text-sm">
"The cloud" isn't magic and isn't weather. It's real servers in real
buildings — you just never touch them.
</div>

---
---

# The NIST definition: five essential characteristics

| Characteristic | In plain words |
|---|---|
| On-demand self-service | You provision it yourself, no ticket to a human |
| Broad network access | Reachable over the network from standard clients |
| Resource pooling | Many customers share the same physical hardware |
| Rapid elasticity | Scale up and down quickly, sometimes automatically |
| Measured service | Usage is metered — and billed |

<div v-click class="mt-6 text-sm opacity-70">
Source: NIST SP 800-145. A rented server you have to email someone about
isn't "cloud" by this definition — self-service and metering are the point.
</div>

---
---

# CapEx vs OpEx

<div class="grid grid-cols-2 gap-8 mt-4">
<div>

### On-premises — CapEx

<v-clicks>

- Pay up front for hardware
- Guess capacity years ahead
- Over-buy → idle servers
- Under-buy → outage on launch day

</v-clicks>

</div>
<div>

### Cloud — OpEx

<v-clicks>

- Pay monthly for what you use
- Start small, grow when needed
- Experiments are cheap to throw away
- But: forgotten resources keep billing

</v-clicks>

</div>
</div>

<div v-click class="mt-8 p-4 rounded bg-red-500/10 text-sm">
OpEx is a double-edged sword. Nobody approves a cloud bill in advance —
it just arrives. That's why Block 4 exists.
</div>

---
---

# Service models: who manages what

| Layer | On-prem | IaaS | PaaS | SaaS |
|---|---|---|---|---|
| Application | You | You | You | Provider |
| Data | You | You | You | Provider* |
| Runtime / middleware | You | You | Provider | Provider |
| OS | You | You | Provider | Provider |
| Virtualization / servers | You | Provider | Provider | Provider |
| Network / building | You | Provider | Provider | Provider |

<div v-click class="mt-4 text-sm opacity-70">
* With SaaS you still own <b>your</b> data and who can access it — the
provider runs everything underneath.
</div>

---
---

# Service models: examples

<v-clicks>

- **IaaS** — you get a virtual machine, you install everything on it.
  AWS EC2, Google Compute Engine, Azure VMs.
- **PaaS** — you bring code or a container, the platform runs it.
  AWS Elastic Beanstalk, Heroku, Google App Engine.
- **SaaS** — you just use the finished product.
  Gmail, GitHub, Google Docs, Slack.

</v-clicks>

<div v-click class="mt-8 p-4 rounded bg-blue-500/10 text-sm">
The trade-off is always the same: the higher you go, the less you manage —
and the less you control.
</div>

---
---

# Deployment models

<v-clicks>

- **Public cloud** — shared provider infrastructure (AWS, Azure, Google
  Cloud). What this course uses.
- **Private cloud** — cloud-style self-service, but on infrastructure
  dedicated to one organization (often on-prem, e.g. OpenStack).
- **Hybrid cloud** — on-prem and public cloud connected and used together.
  Common in banks and government.
- **Multi-cloud** — more than one public provider, on purpose.

</v-clicks>

<div v-click class="mt-6 text-sm opacity-70">
NIST also lists a <b>community</b> cloud — shared by organizations with
common requirements. You'll see it in textbooks more than in job ads.
</div>

---
---

# The shared responsibility model

<div class="grid grid-cols-2 gap-8 mt-4">
<div class="p-4 rounded bg-blue-500/10">

### AWS: security **of** the cloud

- Data centers and physical security
- Hardware, network, hypervisor
- The managed services themselves

</div>
<div class="p-4 rounded bg-green-500/10">

### You: security **in** the cloud

- Who can log in (IAM, MFA)
- Firewall rules (security groups)
- OS patches on your EC2 instances
- Your data: encryption, backups, public or not

</div>
</div>

<div v-click class="mt-8 text-lg font-bold">
Most cloud breaches in the news are on the customer side: a public S3
bucket, a leaked access key, a wide-open firewall rule.
</div>

---
---

# The line moves with the service model

<v-clicks>

- **EC2 (IaaS)** — you patch the OS, configure the firewall, install the
  runtime. A lot is yours.
- **RDS (managed database)** — AWS patches the database engine and OS; you
  still own users, network access, and backups settings.
- **S3 (managed storage)** — AWS runs the storage; you decide who can read
  each bucket.

</v-clicks>

<div v-click class="mt-8 p-4 rounded bg-red-500/10 text-sm">
"It's managed" never means "it's secured for me". Access control is
always your job.
</div>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
Your EC2 server gets hacked through an unpatched package in Ubuntu. Is
that AWS's fault or yours?
</div>

<div v-click class="mt-8 text-lg opacity-70">
Yours — on IaaS, the guest operating system and everything installed on it
are the customer's side of the shared responsibility model.
</div>

---
layout: section
transition: slide-left
---

# Block 2
## AWS global infrastructure

---
---

# Regions

<v-clicks>

- A **region** is a separate geographic area — e.g. `eu-central-1`
  (Frankfurt), `ap-south-1` (Mumbai), `us-east-1` (N. Virginia).
- Regions are isolated from each other. Resources you create live in
  **one** region unless you copy them.
- Most services are regional: an EC2 instance in Frankfurt doesn't show
  up when the console is set to Mumbai.

</v-clicks>

<div v-click class="mt-8 p-4 rounded bg-blue-500/10 text-sm">
"Where did my server go?" — 9 times out of 10, the console's region
selector (top-right) is set to a different region.
</div>

---
---

# Availability Zones

<v-clicks>

- Each region has several **Availability Zones (AZs)** — usually three or
  more — named like `eu-central-1a`, `eu-central-1b`, `eu-central-1c`.
- An AZ is one or more data centers with independent power, cooling, and
  networking, physically separated from the other AZs.
- AZs in a region are linked with low-latency private networking.

</v-clicks>

<div v-click class="mt-8 text-lg font-bold">
Run two copies of your app in two AZs, and a fire or power failure in one
building doesn't take you down. That's the basis of high availability.
</div>

---
---

# Edge locations

<v-clicks>

- Separate from regions: a large network of smaller sites close to users.
- Used by **CloudFront** (CDN — caches your static content near users) and
  **Route 53** (DNS).
- You don't run servers there. They speed up delivery of what lives in
  your region.

</v-clicks>

```text
Region  ⊃  Availability Zones  ⊃  data centers
Edge locations: separate, many, close to users
```

---
---

# Choosing a region: four questions

<v-clicks>

1. **Latency** — how far is it from your users?
2. **Compliance** — does the law or the customer require data to stay in a
   specific country?
3. **Service availability** — not every service or instance type exists in
   every region.
4. **Price** — the same service can cost different amounts in different
   regions.

</v-clicks>

<div v-click class="mt-6 text-sm opacity-70">
Check the current list and prices yourself: the AWS Global Infrastructure
page and the per-service pricing pages. They change — don't trust old
slides, including these.
</div>

---
---

# What about Kazakhstan?

<v-clicks>

- There is **no AWS region in Kazakhstan** or elsewhere in Central Asia
  (as of this lecture).
- Regions people here commonly pick: **Frankfurt** (`eu-central-1`),
  **Stockholm** (`eu-north-1`), **Mumbai** (`ap-south-1`), and the Middle
  East regions (`me-central-1` UAE, `me-south-1` Bahrain).
- Some newer regions are **opt-in** — disabled until you enable them in
  account settings.
- Data-localization rules for personal data can force some systems to stay
  in local data centers — that's a compliance question, not a technical one.

</v-clicks>

<div v-click class="mt-4 p-4 rounded bg-blue-500/10 text-sm">
For this course: use <code>eu-central-1</code> (Frankfurt) unless told
otherwise, so everyone's screenshots look the same.
</div>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
You run your app on one EC2 instance in <code>eu-central-1a</code>. The
whole AZ loses power. What would have kept the app up?
</div>

<div v-click class="mt-8 text-lg opacity-70">
A second instance in another AZ of the same region (e.g.
<code>eu-central-1b</code>) behind a load balancer — AZs fail
independently.
</div>

---
layout: section
transition: slide-left
---

# Block 3
## A tour of the core services

---
---

# The five services this course is built on

| Service | What it is | Week |
|---|---|---|
| <logos-aws-iam /> **IAM** | Users, groups, roles, permissions | 6 (today) |
| <logos-aws-ec2 /> **EC2** | Virtual machines | 7 |
| <logos-aws-vpc /> **VPC** | Your private network in AWS | 8 |
| <logos-aws-s3 /> **S3** | Object storage (files, backups, static sites) | 9 |
| <logos-aws-rds /> **RDS** | Managed relational databases | 9 |

<div v-click class="mt-6 text-sm opacity-70">
AWS has hundreds of services. You need about five to deploy a real app —
start there and ignore the rest of the console for now.
</div>

---
---

# IAM — who can do what

<v-clicks>

- **Users** — a person or app with long-term credentials.
- **Groups** — attach permissions once, add users to the group.
- **Roles** — temporary permissions an EC2 instance, a service, or a
  person can *assume*. Preferred over long-lived keys.
- **Policies** — JSON documents that allow or deny actions on resources.
- IAM is **global** — not tied to a region.

</v-clicks>

<div v-click class="mt-6 p-4 rounded bg-blue-500/10 text-sm">
Rule of thumb: <b>least privilege</b> — give exactly the permissions a
task needs, nothing more.
</div>

---
---

# EC2, VPC, S3, RDS — one sentence each

<v-clicks>

- **EC2** — rent a Linux or Windows VM by the second/hour; you pick size,
  OS image (AMI), disk, and firewall rules (security groups).
- **VPC** — an isolated virtual network: subnets, route tables, internet
  gateway. Every EC2 instance lives in one.
- **S3** — store any number of files ("objects") in "buckets"; private by
  default. Great for backups and static websites.
- **RDS** — PostgreSQL/MySQL/etc. where AWS handles installs, patching, and
  automated backups.

</v-clicks>

---
---

# How they fit together

```text
                 Internet
                    │
        ┌───────────┴─────── VPC (eu-central-1) ───────┐
        │                                              │
        │   public subnet            private subnet    │
        │   ┌──────────┐             ┌──────────┐      │
        │   │   EC2    │ ──────────► │   RDS    │      │
        │   │  (app)   │             │   (DB)   │      │
        │   └────┬─────┘             └──────────┘      │
        └────────┼─────────────────────────────────────┘
                 ▼
            S3 bucket (uploads, backups)

   IAM decides who/what may touch each of these
```

<div v-click class="mt-2 text-sm opacity-70">
By week 9 you'll have built exactly this.
</div>

---
layout: section
transition: slide-left
---

# Block 4
## Account safety & cost control

---
---

# The root user

<v-clicks>

- The email you signed up with is the **root user** — it can do
  *everything*, including closing the account. Its permissions can't be
  limited inside the account.
- Use it only for a handful of account-level tasks (billing settings,
  closing the account).
- **Enable MFA on root first** — before anything else. AWS now requires
  MFA for root on many accounts; do it before you're forced to.
- Never create access keys for the root user.

</v-clicks>

---
---

# Day-one checklist

<v-clicks>

1. Enable **MFA on the root user**
2. Create an **admin identity** for daily work (an IAM user in an
   `Admins` group, or IAM Identity Center) — and enable MFA on it too
3. Sign out of root. Sign in as the admin identity from now on
4. Create a **budget** in AWS Budgets — e.g. $1/month with an email alert
5. Pick your region (`eu-central-1`) and stay in it

</v-clicks>

<div v-click class="mt-6 p-4 rounded bg-blue-500/10 text-sm">
This is literally today's practice, Track A.
</div>

---
---

# Free tier pitfalls

<v-clicks>

- AWS changed its free tier for **new accounts created after July 15,
  2025**: a free plan with sign-up credits for a limited period instead of
  the old 12-month offers. Read the current terms on `aws.amazon.com/free`.
- "Free" usually has limits: instance size, hours per month, GB stored.
- Things that bill even when "nothing is running":
  - **Stopped** EC2 instances still pay for their EBS disks
  - Elastic IPs, NAT Gateways, load balancers, snapshots
  - Resources left in **another region** you forgot about

</v-clicks>

---
---

# Stop vs terminate vs delete

| Action | What happens | Still billing? |
|---|---|---|
| **Stop** an EC2 instance | VM shuts down, disk kept | Yes — for the disk |
| **Terminate** an instance | VM and (by default) root disk deleted | No |
| **Delete** an S3 bucket | Bucket must be empty first | No |
| Leave it "for later" | Nothing happens | **Yes** |

<div v-click class="mt-6 p-4 rounded bg-red-500/10 text-lg font-bold">
End every practice session with cleanup. Check every region you used.
</div>

---
---

# Budgets and the Pricing Calculator

<v-clicks>

- **AWS Budgets** — set a monthly amount; get an email when actual or
  forecasted spend crosses a threshold. A budget is an alarm, **not** a
  hard limit — AWS won't stop your resources for you.
- **Billing / Cost Explorer** — see what you actually spent, by service and
  by region.
- **AWS Pricing Calculator** (`calculator.aws`) — estimate monthly cost
  *before* you build. Works without an account or login.

</v-clicks>

<div v-click class="mt-6 text-sm opacity-70">
Habit for the rest of the course: estimate first, build second, check the
bill third.
</div>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
You set a $1 budget. You forget an instance running and usage reaches $5.
Does AWS stop the instance at $1?
</div>

<div v-click class="mt-8 text-lg opacity-70">
No — a basic budget only sends alerts. Stopping things is still your job
(budget <i>actions</i> can automate some responses, but you have to set
them up).
</div>

---
layout: section
transition: slide-left
---

# Block 5
## Console vs CLI

---
---

# Two ways to drive AWS

<div class="grid grid-cols-2 gap-8 mt-4">
<div>

### Console (web UI)

<v-clicks>

- Great for exploring and learning
- Shows what options exist
- Hard to repeat exactly
- Screenshots are your only record

</v-clicks>

</div>
<div>

### CLI (`aws ...`)

<v-clicks>

- Repeatable — commands can be scripted
- Reviewable — commands can go in Git
- Same API the console uses
- The step before Terraform (week 14)

</v-clicks>

</div>
</div>

<div v-click class="mt-8 text-sm opacity-70">
DevOps habit: learn in the console, then do it again from the CLI.
</div>

---
---

# Setting up the CLI

```bash{1|2|3-7|8}
aws --version                       # AWS CLI v2
aws configure --profile css4003
# AWS Access Key ID [None]:     AKIA...
# AWS Secret Access Key [None]: ****
# Default region name [None]:   eu-central-1
# Default output format [None]: json
aws sts get-caller-identity --profile css4003
```

<v-clicks>

- Credentials go to `~/.aws/credentials`, settings to `~/.aws/config`.
- `sts get-caller-identity` answers "who am I logged in as?" — run it
  first whenever something says "access denied".

</v-clicks>

<div v-click class="mt-4 p-4 rounded bg-red-500/10 text-sm">
Access keys are passwords. Never commit <code>~/.aws/credentials</code> —
the same "secret in history" problem from last week, with a bill attached.
</div>

---
---

# Profiles

```bash
aws s3 ls --profile css4003
export AWS_PROFILE=css4003          # use it for the whole shell session
aws ec2 describe-regions --query 'Regions[].RegionName' --output text
aws ec2 describe-availability-zones --region eu-central-1 \
  --query 'AvailabilityZones[].ZoneName' --output text
```

<v-clicks>

- A **profile** is a named set of credentials + region. Keep one per
  account (personal, course, work).
- `--query` filters the JSON output (JMESPath); `--output text|table|json`
  changes the format.

</v-clicks>

---
layout: section
transition: slide-left
---

# Block 6
## Today's practice

---
---

# Today's practice — First steps in AWS

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

**Track A — you have an AWS account**

1. Root MFA, admin IAM user + MFA
2. $1 monthly budget with an email alert
3. CLI profile + `get-caller-identity`
4. List regions and AZs from the CLI
5. Clean up

</div>
<div>

**Track B — no AWS account**

1. Pricing Calculator estimate (no login)
2. AWS CLI against a local mock (moto)
3. Same CLI moves: identity, regions, IAM user, budget
4. Service-model and shared-responsibility tasks

</div>
</div>

<div v-click class="mt-6 opacity-70 text-sm">
Either track: submit a report with commands, outputs, screenshots, and
answers. Instructions in <code>practices/06-intro-cloud-aws/</code>.
</div>

---
---

# By the end of this lesson, you should be able to

<v-clicks>

- [ ] Explain cloud computing using the five NIST characteristics
- [ ] Classify a service as IaaS, PaaS, or SaaS and say who manages what
- [ ] Draw the line between AWS's and your responsibilities for EC2, RDS, S3
- [ ] Explain regions, AZs, and edge locations, and justify a region choice
- [ ] Secure a new AWS account and set a budget alert
- [ ] Configure an AWS CLI profile and check who you're logged in as

</v-clicks>

---
layout: default
---

# Before next lecture

- [ ] Finish today's practice if you didn't wrap it up in session
- [ ] Track A: double-check every region for leftover resources
- [ ] Install the AWS CLI v2 if you haven't — next week is all EC2

<div class="mt-8 text-sm opacity-60">
Bring a laptop with a working terminal and SSH client next week.
</div>

---
layout: end
---

# Next lecture

AWS EC2 — launching, connecting to, and securing your first virtual
machine.
