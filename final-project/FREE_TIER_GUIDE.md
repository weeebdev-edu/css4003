# Free Tier Guide: running the final project on AWS or GCP without a surprise bill

This guide covers both tracks of the final project, **AWS** and **GCP**. It shows how to sign up, which free allowances matter for this project, how to build the project so it fits inside them, and how to avoid the usual billing traps.

> **Terms change.** Every number below was checked on the official pages on **2026-11-06**. Before you rely on a number, open the link next to it and check it again. If this guide and the official page disagree, the official page wins.

---

## 1. Which track should I choose?

| Your situation | Choose | Why |
|---|---|---|
| You have an AWS account created **on or after 15 July 2025** with credits left | **AWS (own account)** | New accounts get credits and a 6-month Free plan (see §4.1) |
| You have an older AWS account that is still inside its 12-month free tier | **AWS (own account)** | Classic free tier (t2.micro/t3.micro etc.); check what's left in *Billing → Free Tier* |
| You never had a paid Google Cloud / Maps / Firebase account and never used the GCP trial | **GCP (own account)** | $300 Free Trial credit for 90 days, plus Always Free (see §4.2) |
| Both free offers are already used up, or you can't add a card | **AWS Academy Learner Lab** (AWS track) | Course-provided sandbox with its own credit; no card needed |
| You already used everything above | Talk to the instructor **early** | See §7 |

You may switch tracks only before you start building. Mixing clouds in one project is allowed only where the assignment says so (e.g. GitHub Actions + one cloud).

---

## 2. Sign-up, step by step

### 2.0 Before you start (both clouds)

- **Use your own card, in your own name.** Never use a parent's, friend's or company card. If something goes wrong, they get the bill.
- Kazakhstani bank cards (including virtual and debit cards) usually work, but sign-up places a **small temporary verification hold** that is released later. Make sure the card is enabled for online and foreign payments and has a small balance. If the card is declined, try a virtual card from your bank app.
- Use your **personal** email, not a shared one. Turn on **MFA** right away.
- Create **budgets and alerts first, before any resource**. This is the most important step in this guide.

### 2.1 AWS (own account)

1. Go to <https://aws.amazon.com/free/> → **Create a free account**. Choose the **Free plan** if asked. On the Free plan you are not charged; the account closes when the plan ends unless you upgrade (§4.1).
2. Sign in as **root**, then **Security credentials → Assign MFA device** (authenticator app).
3. **Billing and Cost Management → Budgets → Create budget**:
   - a **zero-spend budget** (alerts at any charge), and
   - a **monthly cost budget** set to your variant's budget (or a small amount like $5), with alerts at 50 / 80 / 100%.
4. **Billing preferences → turn on Free Tier usage alerts** (email when you approach a free-tier limit).
5. Create an **IAM admin user/role for daily work** with MFA, and stop using root.
6. Choose one region and stay in it, e.g. **`eu-central-1` (Frankfurt)** for latency from Almaty, or the region your variant says. Make sure every service you need exists there.
7. Bookmark **Cost Explorer** and **Billing → Free Tier**. You'll check them every day you work (§6).

### 2.2 GCP (own account)

1. Go to <https://cloud.google.com/free> → **Get started for free**. Sign in with your personal Google account, enter your address and card. Google uses the card to verify your identity and may also ask you to verify your bank account.
2. You get a **Free Trial billing account with $300 credit for 90 days**. Billing is **not** switched to paid automatically (§4.2).
3. Create a project for the course, e.g. `<resource_prefix>-final`.
4. **Billing → Budgets & alerts → Create budget** for the project. Use your variant budget (or a small amount), with alert thresholds at 50 / 90 / 100%.
5. Turn on **2-Step Verification** on the Google account. In IAM, grant others only what they need (no Owner for automation; use a service account).
6. Choose a region:
   - For **Always Free** compute and storage you must use **`us-central1`, `us-east1` or `us-west1`** (§4.2).
   - For trial-credit resources you may use `europe-west3` (Frankfurt) or another region. Keep everything in one region.
7. Bookmark **Billing → Reports** and the **Free Trial status / credit balance** page.

### 2.3 AWS Academy Learner Lab

- No sign-up and no card: you get access from the course in AWS Academy.
- Things to know **per the Learner Lab Readme** (read it in the lab):
  - The lab has a **limited credit**. The remaining budget is shown in the lab; when it's gone, the lab account is disabled.
  - A **session lasts a few hours** (shown as a timer). When it ends, running EC2 instances are **stopped** and restarted the next time you start the lab. Your data is kept.
  - Only certain regions are allowed (usually **`us-east-1`** and **`us-west-2`**) and only a subset of services.
  - You **cannot create IAM users or roles**. Use the provided **`LabRole` / `LabInstanceProfile`**.
  - CLI credentials are **temporary**: copy them from *AWS Details* each session.
- Because instances stop between sessions, **design for restart**: services must start on boot (systemd / k3s), and nothing important may live only in memory.

---

## 3. A free-tier-friendly architecture for this project

The project needs Kubernetes, a self-hosted database, object storage, secrets, monitoring, logging, IaC and CI/CD. Here is how to fit that into free allowances and credits.

**Recommended baseline (both clouds):**

```
                 Internet
                    │  (80/443 only)
            ┌───────▼────────┐
            │  VM #1 (k3s    │  ← public subnet, 1 public IP
            │  server+Traefik│     app, Prometheus, Grafana, Loki
            │  ingress)      │
            └───────┬────────┘
                    │ private traffic
            ┌───────▼────────┐
            │  VM #2 (k3s    │  ← optional 2nd node (variant: k3s-2-nodes)
            │  agent)        │     DB (StatefulSet + PVC) or app replicas
            └────────────────┘
   Object storage: S3 / GCS bucket      Secrets: SSM Parameter Store / Secret Manager
   Images: ECR / Artifact Registry      State: Terraform (local or bucket backend)
```

- **Kubernetes = k3s on 1–2 small VMs.** A managed control plane is not free: EKS costs **$0.10 per cluster per hour** ([EKS pricing](https://aws.amazon.com/eks/pricing/)). GKE has a free tier for one cluster's management fee (§4.2), but nodes still cost money. Use managed Kubernetes only if your variant says `managed-k8s-if-credits` and you watch the credits closely.
- **VM size:** k3s + PostgreSQL/MySQL + Prometheus/Grafana/Loki needs more than 1 GB of RAM.
  - The always-free e2-micro (GCP) and t3.micro/t4g.micro (AWS) have 1 GB, which is enough only for a single light piece (e.g. the DB node).
  - Plan **~2 GB or more** for the main node: t3.small / t4g.small on AWS, e2-small / e2-medium on GCP. Keep in mind this uses credits.
- **Database: self-hosted in the cluster** (StatefulSet + PVC) **or on VM #2**, never a managed DB (the assignment forbids it, and it eats credits).
- **No NAT gateway.** Give the one public VM a public IP and keep VM #2 private, with egress only when needed. Or put both in a public subnet, lock them down with security groups / firewall rules, and explain the trade-off in your report.
- **Load balancer only for the demo.** Use the k3s ingress (Traefik) on the public VM day to day. If your design needs a cloud LB, create it with Terraform for the demo and destroy it afterwards.
- **Monitoring and logging inside the cluster** (Prometheus + Grafana + Loki/Alloy): no cloud bill. If your variant is `cloud-logging-fluentbit`, keep log volume small (§5).
- **Secrets:**
  - On AWS, use **SSM Parameter Store standard parameters**: no additional charge ([Systems Manager pricing](https://aws.amazon.com/systems-manager/pricing/)). Secrets Manager costs $0.40 per secret per month ([pricing](https://aws.amazon.com/secrets-manager/pricing/)), but it's fine for 1–3 secrets on credits.
  - On GCP, Secret Manager's free tier is 6 active secret versions and 10,000 access operations per month.
- **Terraform state:** a local file plus a careful `.gitignore`, or a bucket backend. **Never commit state:** it can contain secrets.

---

## 4. Free allowances that matter for this project

> Checked 2026-11-06. **Verify on the official page — terms change.**

### 4.1 AWS

**Accounts created on or after 15 July 2025** ([AWS Free Tier](https://aws.amazon.com/free/), [Free Tier FAQs](https://aws.amazon.com/free/free-tier-faqs/)):

| Item | What you get |
|---|---|
| Credits | **$100 at sign-up** plus **up to $100 more** for completing activities in the *Explore AWS* widget, so **up to $200** |
| Free plan length | **Up to 6 months** or until credits run out, whichever comes first |
| When the Free plan ends | AWS **suspends** the account, keeps your data for **90 days**, then **permanently deletes** it unless you upgrade to a Paid plan |
| Charges | None on the Free plan unless you upgrade. Usage above always-free limits is paid from your credits |
| Always free | "30+ services" with monthly always-free limits (see each service's page) |
| EC2 free-tier eligible types | **t3.micro, t3.small, t4g.micro, t4g.small, c7i-flex.large, m7i-flex.large** ([EC2 free tier usage](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-free-tier-usage.html)) |

**Older accounts (before 15 July 2025):** the classic 12-month free tier; eligible EC2 types are **t2.micro and t3.micro** (same EC2 docs page). Check what's left in *Billing → Free Tier*.

**Per-service limits relevant here:**

| Service | Free allowance / price to know | Source |
|---|---|---|
| EC2 | Eligible types above; check usage in *Billing → Free Tier* | [EC2 docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-free-tier-usage.html) |
| T4g trial | Until **31 Dec 2026**: **750 free hours/month of t4g.small** for all customers (surplus CPU credits still charged) | [EC2 docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-free-tier-usage.html) |
| Public IPv4 | **$0.005 per IP per hour**, in use or idle, since 1 Feb 2024. Free tier includes **750 hours/month** of in-use public IPv4 for EC2 free-tier customers | [VPC pricing](https://aws.amazon.com/vpc/pricing/), [announcement](https://aws.amazon.com/about-aws/whats-new/2024/02/aws-free-tier-750-hours-free-public-ipv4-addresses/) |
| NAT Gateway | **$0.045/hour + $0.045/GB** processed (us-east-1), **not free** | [VPC pricing](https://aws.amazon.com/vpc/pricing/) |
| ALB | **$0.0225/hour + $0.008 per LCU-hour** (us-east-1). Free tier: **750 h/month** shared between Classic and Application LBs + **15 LCUs** | [ELB pricing](https://aws.amazon.com/elasticloadbalancing/pricing/) |
| EKS | **$0.10 per cluster per hour**, not free | [EKS pricing](https://aws.amazon.com/eks/pricing/) |
| ECR | **500 MB/month** private storage for one year (free tier), 50 GB always free for public repos | [ECR pricing](https://aws.amazon.com/ecr/pricing/) |
| SSM Parameter Store | Standard parameters: **no additional charge** (standard throughput) | [SSM pricing](https://aws.amazon.com/systems-manager/pricing/) |
| Secrets Manager | **$0.40/secret/month + $0.05 per 10,000 API calls**. No separate trial; credits apply | [Secrets Manager pricing](https://aws.amazon.com/secrets-manager/pricing/) |
| CloudWatch | **10** custom/detailed metrics, **10** alarm metrics, **3** custom dashboards, **5 GB** logs (ingestion + archive + Logs Insights scanned), 1M API requests | [CloudWatch pricing](https://aws.amazon.com/cloudwatch/pricing/) |
| S3, EBS | Check the current always-free / free-plan amounts on each pricing page; they're billed per GB-month beyond that | [S3 pricing](https://aws.amazon.com/s3/pricing/), [EBS pricing](https://aws.amazon.com/ebs/pricing/) |

### 4.2 GCP

Source: [Free Google Cloud features and trial offer](https://docs.cloud.google.com/free/docs/free-cloud-features).

**Free Trial:**

| Item | What you get |
|---|---|
| Credit | **$300 Welcome credit, valid for 90 days** |
| Eligibility | Never a paying user of Google Cloud, Google Maps Platform or Firebase, and never signed up for the Free Trial before |
| When it ends | Resources are **stopped**, data in e.g. Compute Engine is **marked for deletion**, and you lose access. You have a **30-day grace period** to upgrade and maybe recover; after that, resources are **permanently deleted**. **No automatic charging**: you must upgrade manually |
| Not allowed during the trial | GPUs, Windows Server VMs, Marketplace, **quota increases**, crypto mining, VMware Engine, using the credit for the Gemini API / generative AI partner models |

**Always Free (also after the trial):**

| Service | Always Free allowance |
|---|---|
| Compute Engine | **1 non-preemptible e2-micro** VM/month in **us-west1, us-central1 or us-east1**; **30 GB-months standard persistent disk**; **1 GB** egress from North America per month |
| Cloud Storage | **5 GB-months regional storage** (US regions only: us-east1, us-west1, us-central1); **5,000 Class A** and **50,000 Class B** operations; **100 GB** egress from North America per month |
| GKE | **One free Autopilot or zonal Standard cluster per month**. The credit covers the **cluster management fee only**; nodes and pods are still charged |
| Secret Manager | **6 active secret versions** and **10,000 access operations** per month |
| Artifact Registry | **0.5 GB** storage per month |
| Cloud Logging | First **50 GiB** of logs per project per month |
| Cloud Monitoring | First **1 million** time series returned by API read calls per billing account |
| Cloud Build | **2,500 build-minutes/month** (e2-standard-2) |
| Cloud Run | **2 million requests**, **360,000 GB-seconds** memory, **180,000 vCPU-seconds** per month |

Other prices to know:
- **External IPv4 on a VM: $0.005 per hour** ([VPC network pricing](https://cloud.google.com/vpc/network-pricing)). Check that page for any free hours.
- **Cloud NAT** and **load balancer forwarding rules** are charged hourly ([Cloud NAT pricing](https://cloud.google.com/nat/pricing), [network pricing](https://cloud.google.com/vpc/network-pricing)). Check the current numbers on those pages.

---

## 5. Cost traps and how to avoid them

| Trap | Why it costs | What to do |
|---|---|---|
| **NAT gateway / Cloud NAT** | Hourly fee plus per-GB processing, even when idle (AWS: $0.045/h + $0.045/GB) | Don't create one. Use a public VM with a tight SG/firewall, or create NAT only for the demo and destroy it |
| **Public IPv4 addresses** | AWS and GCP both charge per hour per IP, **including idle Elastic IPs** | One public IP per project; release unused Elastic IPs / static IPs |
| **Load balancers** | Charged per hour as long as they exist, with no traffic needed | Use k3s ingress daily; create the LB only for the demo, then `terraform destroy` it |
| **Managed Kubernetes** | EKS $0.10/h per cluster; GKE nodes cost money | k3s on VMs |
| **Managed databases** | Not allowed in this project, and expensive | Self-hosted DB |
| **Forgotten disks and snapshots** | Billed per GB-month after the VM is gone | `terraform destroy`; check *EBS → Volumes / Snapshots* or *Compute Engine → Disks / Snapshots* |
| **Log volume** | CloudWatch free tier is 5 GB, Cloud Logging 50 GiB/project; debug logs in a loop fill that fast | `LOG_LEVEL=INFO`, short retention (e.g. 7 days), no logging of request bodies |
| **Data egress** | Traffic leaving the cloud is billed beyond the free amount | Don't serve large files publicly; keep backups in the same region |
| **Leftover clusters / VMs in other regions** | You forget them because the console shows one region at a time | Use one region; check *Billing → by region* or *Cost Explorer* |
| **Instance types outside the free tier** | Larger types burn credits quickly | Stick to the types in §4; read the price before `apply` |
| **Secrets Manager with many secrets** | $0.40 per secret per month each | Few secrets, or Parameter Store standard (AWS) / within 6 versions (GCP) |
| **Terraform state or `.env` in git** | Leaked keys get abused within minutes, and the bill is yours | `.gitignore` state and `.env`; gitleaks in CI; rotate any key that leaked |

---

## 6. Daily routine (5 minutes)

**When you start working:**
1. Open **Billing / Cost Explorer** (AWS) or **Billing → Reports** (GCP). Check yesterday's cost and the credit balance.
2. Learner Lab: **Start Lab** and copy fresh credentials from *AWS Details*.

**When you stop working:**
1. **Scale to zero or stop:** stop the VMs, or `kubectl scale deploy --all --replicas=0` if the VMs must keep running.
2. Destroy anything that bills by the hour and isn't needed until next time: LB, NAT, managed cluster.
3. **After every demo or test run:** `terraform destroy` (or `terraform destroy -target=...` for the expensive parts). A full `destroy` and then `apply` from scratch is also proof that your IaC works.
4. Once a week, look for leftovers: unattached disks, old snapshots, unused IPs, empty buckets, images in the registry.

---

## 7. If your credits run out

1. **Tell the instructor immediately**, before the deadline rather than after.
2. **Switch to AWS Academy Learner Lab** (AWS track) and re-deploy with your Terraform. That's why everything must be in code.
3. **Record your demo before you tear things down.** A 3–5 minute video showing the running system (app, Grafana, logs, CI run, failover drill) is part of the submission anyway.
4. **Local cluster as a last resort:** the **orchestration** part only may be demonstrated on a local **kind / k3d** cluster, **with the point cap given in the RUBRIC**. Cloud networking, IAM/roles, secret manager and IaC must still be shown on a real cloud, at least once, recorded.

---

## 8. Checklist before the defense

- [ ] Budget and alerts exist and you can show them (screenshot in the report)
- [ ] MFA on root / account owner
- [ ] Everything you create is in one region and visible in Terraform
- [ ] `terraform apply` from scratch works (tested this week)
- [ ] Demo video recorded (in case the live demo fails or credits run out)
- [ ] No NAT, LB or extra public IPs left running when you're not demoing
- [ ] No secrets in git (`gitleaks` passes), no state file committed
- [ ] Cost estimate and actual spend (screenshot) in the report
- [ ] Plan for after the defense: `terraform destroy`, delete leftover disks/snapshots/images, and (for your own account) decide whether to close it or keep it on the free plan

---

### Sources (checked 2026-11-06)

- AWS Free Tier: <https://aws.amazon.com/free/>, FAQs: <https://aws.amazon.com/free/free-tier-faqs/>
- EC2 free tier usage and eligible types: <https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-free-tier-usage.html>
- VPC pricing (NAT, public IPv4): <https://aws.amazon.com/vpc/pricing/>; public IPv4 free tier: <https://aws.amazon.com/about-aws/whats-new/2024/02/aws-free-tier-750-hours-free-public-ipv4-addresses/>
- ELB pricing: <https://aws.amazon.com/elasticloadbalancing/pricing/>
- EKS pricing: <https://aws.amazon.com/eks/pricing/>
- ECR pricing: <https://aws.amazon.com/ecr/pricing/>
- Systems Manager pricing: <https://aws.amazon.com/systems-manager/pricing/>
- Secrets Manager pricing: <https://aws.amazon.com/secrets-manager/pricing/>
- CloudWatch pricing: <https://aws.amazon.com/cloudwatch/pricing/>
- Google Cloud free features and trial: <https://docs.cloud.google.com/free/docs/free-cloud-features>
- Google Cloud network pricing: <https://cloud.google.com/vpc/network-pricing>; Cloud NAT pricing: <https://cloud.google.com/nat/pricing>
- Google Cloud payment methods: <https://docs.cloud.google.com/billing/docs/how-to/payment-methods>
