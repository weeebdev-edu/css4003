# Practice 06 — First Steps in AWS

**Objective:** get into AWS and drive it from both the console and the
AWS CLI: find out who you are, which region you're in, and which regions
and AZs exist. These are the skills from today's lecture.

**Timebox:** ~50 min of actual work, in class. If you're stuck on one step
for more than 10 minutes, ask instead of pushing through alone.

> **AWS access isn't guaranteed for everyone, so there are two tracks.**
> Do **Track A** if you have access to the **AWS Academy Learner Lab**
> course. Otherwise do **Track B**, which needs no account and no card.
> Tell the instructor which track you did.

> **Using AWS Academy Learner Lab**
>
> 1. Open the **AWS Academy Learner Lab** course → **Modules** →
>    **Launch AWS Academy Learner Lab** → **Start Lab**.
> 2. Wait until the dot next to **AWS** turns green (1–3 min), then click
>    **AWS** to open the console.
> 3. Region: **`us-east-1`**. Per the Learner Lab Readme, only `us-east-1`
>    and `us-west-2` are allowed.
> 4. CLI credentials: **AWS Details** → **Show** next to *AWS CLI*. They
>    are temporary (`aws_access_key_id`, `aws_secret_access_key`,
>    `aws_session_token`). Paste them into `~/.aws/credentials` under
>    `[default]`, or use the terminal inside the lab page, where the CLI is
>    already configured.
> 5. Learner Lab is a sandbox: you **cannot** create IAM users, touch the
>    root user or MFA, or create Budgets. Don't try. Those steps are
>    concept-only below.
> 6. A session lasts a few hours and lab credits are limited. Click
>    **End Lab** when you're done.

## Definition of done

- [ ] Track A **or** Track B done, every step shown with the command you
      ran and its output (or a screenshot for console steps)
- [ ] You can answer the questions under "Be ready to explain"
- [ ] Cleanup done (Track A: **End Lab**. Track B: moto stopped)
- [ ] Submitted the way your instructor says. Submitting is also your
      attendance record for this session.

The concept tasks (Part C) are **homework**, not part of the class time.

---

## Track A — AWS Academy Learner Lab

### A1. Start the lab and open the console

1. Start the lab as described in the box above and open the console.
2. Top-right: check the **region selector** says **N. Virginia
   (us-east-1)**.
3. Use the search bar to open **EC2**, **S3** and **IAM** one after
   another. Note which ones show a region in the top-right and which one
   says **Global**.
4. Screenshot: the console with the region selector visible.

### A2. Who am I? (CLI)

Paste the credentials from **AWS Details** into `~/.aws/credentials`, or
use the terminal on the lab page:

```bash
aws --version
aws configure set region us-east-1
aws sts get-caller-identity
```

The `Arn` looks like
`arn:aws:sts::<account>:assumed-role/voclabs/user<…>`. That means you are
**not** an IAM user and not root: you're using **temporary credentials**
of a role (`voclabs`) that AWS Academy lets you assume. That's also why
the session token expires when the lab ends.

### A3. Explore regions and AZs

```bash
aws ec2 describe-regions --query 'Regions[].RegionName' --output text
aws ec2 describe-regions --all-regions \
  --query 'Regions[].[RegionName,OptInStatus]' --output table
aws ec2 describe-availability-zones --region us-east-1 \
  --query 'AvailabilityZones[].[ZoneName,ZoneId,State]' --output table
```

Save the outputs. How many AZs does `us-east-1` have?

### A4. Concept only: securing a real account

You can't do these in Learner Lab, but be ready to explain them. In your
**own** AWS account you would, on day one:

1. Turn on **MFA for the root user**, and never create root access keys.
2. Create an **IAM admin user** (in an `Admins` group with
   `AdministratorAccess`) with its own MFA, and use only that day to day.
3. Create a **$1 monthly budget** with an email alert, because a budget
   only alerts and never stops anything.

### A5. Cleanup

Click **End Lab**. Nothing you did today costs anything, but ending the
lab stops the clock on your session.

---

## Track B — no AWS account

### B1. Estimate a cost with the Pricing Calculator (~10 min)

The AWS Pricing Calculator needs no login: <https://calculator.aws>

1. **Create estimate** → location **US East (N. Virginia)**.
2. Add **Amazon EC2**: 1 Linux instance, `t3.micro`, on-demand, running
   **24 hours a day for a month**, with 8 GB of gp3 EBS storage. Note the
   monthly total.
3. Change the EC2 usage to **8 hours a day, weekdays only** and note the
   new total.
4. Write the two totals down. Why isn't the second one just a third of the
   first?

### B2. Install the tools (~10 min)

You need Python 3 and the AWS CLI. **moto** is an open-source library
that pretends to be AWS on your own machine. Nothing touches the real
cloud and nothing costs money.

```bash
python3 -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install "moto[server]" awscli
aws --version
```

(If you already have the AWS CLI v2 installed, use it and skip `awscli`
in the `pip install`.)

Start the mock in a **separate terminal** and leave it running:

```bash
source .venv/bin/activate
moto_server -p 5000
```

### B3. Configure a CLI profile

moto accepts any credentials. Use fake ones:

```bash
aws configure set aws_access_key_id test --profile mock
aws configure set aws_secret_access_key test --profile mock
aws configure set region us-east-1 --profile mock
aws configure set output json --profile mock
```

Every command from here on needs both flags. Save typing with a
variable:

```bash
E="--profile mock --endpoint-url http://localhost:5000"
aws sts get-caller-identity $E
```

(In zsh, which is the default on macOS, write `${=E}` instead of `$E`, or
just type the flags out each time.)

### B4. Explore regions and AZs

```bash
aws ec2 describe-regions $E --query 'Regions[].RegionName' --output text
aws ec2 describe-availability-zones $E \
  --query 'AvailabilityZones[].ZoneName' --output text
```

> moto mimics the AWS **API**, not all of AWS's behaviour. It won't send
> budget emails or check MFA. Treat it as practice for the commands.

### B5. Concept only: securing a real account

Same as **A4** above: be ready to explain root MFA, an IAM admin user and
a $1 budget.

### B6. Cleanup

Stop `moto_server` with `Ctrl+C`. It keeps everything in memory, so
stopping it wipes all state.

---

## Be ready to explain

1. Your `get-caller-identity` (Track A) shows `assumed-role/voclabs/...`.
   What does that mean, and why is it safer than a long-term access key?
2. What's the difference between a region and an Availability Zone? Why
   would you spread an app across two AZs?
3. Which AWS region would you pick for an app whose users are in Almaty,
   and why? Name at least two factors from the lecture.
4. In your own account: why shouldn't you use the root user day to day,
   and what does a $1 budget actually do when you go over it?
5. You find an access key in a public GitHub repo. In what order do you
   remove it from the repo, deactivate it, and check what it was used
   for? Why that order?

---

## Homework (not in class): Part C, concept tasks

**C1. Classify each one as IaaS, PaaS, or SaaS.** Give one line of
reasoning for each.

| Service | Model | Why |
|---|---|---|
| Amazon EC2 | | |
| Gmail | | |
| AWS Elastic Beanstalk | | |
| GitHub | | |
| Amazon RDS | | |
| A VM you rent from a hosting company and SSH into | | |

**C2. Shared responsibility.** For each task, write **AWS** or **Customer**.

| Task | EC2 | RDS | S3 |
|---|---|---|---|
| Patching the operating system | | | |
| Physical security of the data center | | | |
| Deciding who can access the data | | | |
| Firewall / network access rules | | | |
| Replacing a failed disk | | | |
| Turning on encryption / backups settings | | | |

## Common mistakes

- **"Lab is not ready" / the AWS button doesn't open the console.** Wait
  for the green dot after **Start Lab**. It can take a few minutes.
- **`ExpiredToken` or `InvalidClientTokenId` in the CLI (Track A).** The
  lab session ended or was restarted, so the credentials changed. Copy
  them again from **AWS Details**, including `aws_session_token`.
- **`UnauthorizedOperation` in another region (Track A).** Learner Lab
  only allows `us-east-1` and `us-west-2`. Switch the region back.
- **"I can't see the thing I created."** The console's region selector
  (top-right) is set to a different region. IAM is global, but almost
  everything else is regional.
- **`Could not connect to the endpoint URL` (Track B).** `moto_server`
  isn't running, or it's on a different port from the one in `$E`.
- **The zsh variable doesn't expand (Track B).** In zsh,
  `aws ... $E` passes the whole string as one argument. Use `${=E}` or
  bash.
- **Screenshots leak secrets.** Never show a secret access key or session
  token in a screenshot. Blur them.
