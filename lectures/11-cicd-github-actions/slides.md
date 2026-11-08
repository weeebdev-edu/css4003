---
theme: seriph
title: "CSS4003 — Lecture 11: CI/CD with GitHub Actions"
info: |
  CSS4003 — DevOps 2 · Narxoz University
  Lecture 11 of 15
background: /cover-bg.svg
transition: fade
mdc: true
download: true
---

# CSS4003 — DevOps 2

## Lecture 11: CI/CD with GitHub Actions

<div class="pt-8 opacity-70">
Nauruzbayeva Farikha · Lesson 11
</div>

---
layout: default
---

# Recap — Lesson 10 (Docker in Production)

<v-clicks>

- Tag vs digest? <span v-click class="opacity-60">(a tag is a movable name; a digest is the content hash — same digest, same image, forever)</span>
- Why does the server only pull, never build? <span v-click class="opacity-60">(it must run the exact image that was tested)</span>
- What did Trivy do for us? <span v-click class="opacity-60">(scanned the image for known CVEs and failed the gate on HIGH/CRITICAL)</span>
- How did we roll back? <span v-click class="opacity-60">(changed the tag back to v1 and `docker compose up -d` — the old image was still in the registry)</span>
- What does an orchestrator add? <span v-click class="opacity-60">(scheduling, self-healing, scaling, rolling updates)</span>

</v-clicks>

---
---

# Today's agenda

<v-clicks>

- [ ] CI, continuous delivery, continuous deployment
- [ ] GitHub Actions anatomy: events, jobs, steps, runners, actions
- [ ] One workflow, step by step: lint, tests, matrix, services, artifacts, cache
- [ ] Build and push the image to GHCR with meaningful tags
- [ ] Security gates: gitleaks, Trivy, pip-audit, least-privilege tokens
- [ ] Branch protection: red check = no merge
- [ ] CD: approvals, deploy over SSH, rollback
- [ ] Reuse, self-hosted runners, debugging
- [ ] Live demo → straight into today's practice

</v-clicks>

---
layout: center
class: text-center
---

# The scenario

<div class="text-lg text-left mt-4 max-w-2xl mx-auto">

Last week **you** built the image, ran Trivy, pushed v2 and edited
`.env` on the server. It worked — on a good day. On a bad day someone
forgets the tests, pushes from a laptop with uncommitted changes, or
tags `latest` by mistake.

</div>

<div v-click class="mt-8 text-xl font-bold">
Make every change go through the same steps, automatically: test, scan,
build, push, deploy — and refuse anything that fails.
</div>

---
layout: section
transition: slide-left
---

# Block 1
## CI and CD

---
---

# Three words, three promises

| Term | Promise | Ends with |
|---|---|---|
| **Continuous Integration** | Every change is built and tested together, many times a day | A green or red check |
| **Continuous Delivery** | Every green change *can* be released at any time | A deployable artifact (image) |
| **Continuous Deployment** | Every green change *is* released automatically | Running in production |

<div v-click class="mt-4 text-sm opacity-70">
Most teams do CI + continuous delivery, with a <b>human approval</b>
before production. That's what we build today.
</div>

---
---

# Why bother

<v-clicks>

- **Fast feedback** — a broken test is found in minutes, by a machine, not
  by a user next week.
- **Same steps every time** — no "I forgot to run the linter".
- **Small changes** — merging often keeps conflicts small.
- **Traceability** — every image and deploy points to a commit and a run.
- **Safe releases** — rollback is "deploy the previous tag", not heroics.

</v-clicks>

<div v-click class="mt-6 p-4 rounded bg-blue-500/10 text-sm">
The pipeline is the <b>only</b> way code reaches production. If it's
not in git and green, it doesn't ship.
</div>

---
---

# The pipeline we'll build

```text
pull request ─▶ lint ─▶ test (3.11, 3.12) ─▶ secret scan ─▶ ✅ / ❌  (merge allowed?)
push to main ─▶ … same … ─▶ build ─▶ Trivy ─▶ push ghcr.io/…:sha-…, main
tag v1.2.0   ─▶ … same … ─▶ build ─▶ Trivy ─▶ push ghcr.io/…:1.2.0
manual       ─▶ approve ─▶ deploy 1.2.0 over SSH ─▶ (or roll back to 1.1.0)
```

<div v-click class="mt-4 text-sm opacity-70">
One YAML file for CI and image, one for deploy — both live in the repo,
reviewed like any other code.
</div>

---
layout: section
transition: slide-left
---

# Block 2
## GitHub Actions anatomy

---
---

# Workflow → jobs → steps

```text
.github/workflows/ci.yml        ← a workflow (YAML file)
  on: pull_request, push        ← events that start it
  jobs:
    lint:                        ← a job: runs on its own fresh machine (runner)
      steps:                     ← steps run in order, in the same machine
        - uses: actions/checkout@v7      ← a reusable action
        - run: ruff check .               ← a shell command
    test:                        ← jobs run in parallel unless you add needs:
```

<v-clicks>

- **Workflow** — a YAML file in `.github/workflows/`.
- **Job** — a set of steps on one **runner** (VM). Jobs don't share files.
- **Step** — either `uses:` an action or `run:` a shell command.

</v-clicks>

---
---

# Events: when does it run?

```yaml {1|2|3-5|6|7-8|9-10}
on:
  pull_request:                 # every PR (and every push to it)
  push:
    branches: [main]            # merges to main
    tags: ["v*"]                # release tags like v1.2.0
  workflow_dispatch:            # a "Run workflow" button (with inputs)
  schedule:
    - cron: "0 3 * * 1"         # every Monday 03:00 UTC (e.g. a weekly scan)
  release:
    types: [published]          # when a GitHub Release is published
```

<div v-click class="mt-2 text-sm opacity-70">
Filters (<code>branches</code>, <code>tags</code>, <code>paths</code>)
keep pipelines from running for nothing — e.g. skip CI when only
<code>docs/</code> changed.
</div>

---
---

# Runners

<v-clicks>

- **GitHub-hosted** — a fresh VM per job: `ubuntu-latest`,
  `windows-latest`, `macos-latest`. Docker, git, Python, Node are
  pre-installed. Thrown away after the job.
- Free minutes for public repositories; private repos get a monthly
  quota.
- **Self-hosted** — your own machine runs the jobs (Block 7).
- Every job starts **empty**: no files from the previous job, no
  dependencies — that's why we cache and use artifacts.

</v-clicks>

---
---

# Actions: reuse, but pin

```yaml
- uses: actions/checkout@v7                      # major version tag
- uses: actions/setup-python@v7
- uses: docker/build-push-action@v7
- uses: some-org/some-action@8f4b7f84bd579b95d7f0b90f8d8b6e5d9b4f1d3a  # full SHA
```

<v-clicks>

- An action is code from someone else's repo that runs **with your
  token and secrets**.
- `@v7` follows the latest v7.x — convenient, but the owner can move it.
- `@<full commit SHA>` can never change — the safest choice for
  third-party actions.
- Prefer actions from `actions/`, `docker/` and well-known vendors.

</v-clicks>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
The <code>test</code> job needs the Docker image built by the
<code>build</code> job. Will it find it?
</div>

<div v-click class="mt-8 text-lg opacity-70">
No — every job runs on a fresh runner. Pass files with
<b>artifacts</b> (or push the image to a registry), and order jobs with
<code>needs:</code>.
</div>

---
layout: section
transition: slide-left
---

# Block 3
## One workflow, step by step

---
---

# Start: trigger, permissions, concurrency

```yaml {1|3-7|9-10|12-14}
name: ci

on:
  pull_request:
  push:
    branches: [main]
    tags: ["v*"]

permissions:
  contents: read            # least privilege for every job

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true  # a new push cancels the old run of that branch
```

<div v-click class="mt-2 text-sm opacity-70">
<code v-pre>${{ … }}</code> is an <b>expression</b>: GitHub fills it in before
the job starts (<code>github.ref</code> = the branch or tag).
</div>

---
---

# Job 1: lint

```yaml {2|4|5-9|10|11-12}
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v7
        with:
          python-version: "3.12"
          cache: pip
      - run: pip install -r requirements-dev.txt
      - run: ruff check .
      - run: ruff format --check .
```

<v-clicks>

- `checkout` clones your repo into the runner — without it the runner is
  empty.
- `cache: pip` restores downloaded packages from the last run: faster
  installs.
- Any command that exits non-zero fails the step → the job → the check.

</v-clicks>

---
---

# Job 2: tests on two Python versions

```yaml {3-5|7-9|10-11}
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.11", "3.12"]
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v7
        with: { python-version: "${{ matrix.python-version }}", cache: pip }
      - run: pip install -r requirements-dev.txt
      - run: pytest -q --junitxml=report-${{ matrix.python-version }}.xml
```

<v-clicks>

- The **matrix** creates one job per value: `test (3.11)` and
  `test (3.12)`, running in parallel.
- Add more axes (OS, DB version) → every combination becomes a job.

</v-clicks>

---
---

# A real database in CI: service containers

```yaml {3-11|14-15}
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:17
        env:
          POSTGRES_PASSWORD: test        # throwaway, CI-only
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready -U postgres"
          --health-interval 5s --health-retries 10
    steps:
      - uses: actions/checkout@v7
      - run: pytest -q
        env: { DB_URL: "postgresql://postgres:test@localhost:5432/postgres" }
```

<div v-click class="mt-2 text-sm opacity-70">
The runner starts Postgres <b>before</b> the steps and waits until the
health check passes. Integration tests hit a real DB, then it's thrown
away.
</div>

---
---

# Artifacts and test reports

```yaml {1-2|3-5}
      - uses: actions/upload-artifact@v7
        if: always()              # upload even when tests failed
        with:
          name: test-report-${{ matrix.python-version }}
          path: report-${{ matrix.python-version }}.xml
```

<v-clicks>

- **Artifacts** — files saved from a run: test reports, coverage, built
  binaries. Download them from the run's Summary page.
- Another job can fetch them with `actions/download-artifact`.
- `if: always()` — a failed run is exactly when you want the report.
- Artifacts expire (90 days by default) — they're not a registry.

</v-clicks>

---
---

# Cache vs artifact

| | Cache | Artifact |
|---|---|---|
| Purpose | Speed up the **next** run | Keep **this** run's output |
| Example | pip downloads, Docker layers | Test report, coverage, binary |
| Key | Hash of `requirements*.txt` | Name you choose |
| Missing? | Run is slower, still correct | Something downstream breaks |

<div v-click class="mt-4 text-sm opacity-70">
A cache must never change the result — only the time. If a stale cache
breaks your build, the cache key is wrong.
</div>

---
---

# Ordering jobs with `needs` and `if`

```yaml {1-3|4}
  image:
    needs: [lint, test, secrets-scan]   # wait for all three, all must be green
    if: github.event_name == 'push'      # not on pull requests
    runs-on: ubuntu-latest
```

<v-clicks>

- Without `needs:` jobs run in **parallel**.
- `needs:` = "start after these succeeded" — one red job stops the rest.
- `if:` skips a job (shown as grey "skipped"): here we build and push
  images only for `main` and tags, never for PRs.

</v-clicks>

---
layout: section
transition: slide-left
---

# Block 4
## Build and push the image

---
---

# Logging in to GHCR from Actions

```yaml {1-4|6-11}
    permissions:
      contents: read
      packages: write          # only this job may push packages
    steps:
      - uses: actions/checkout@v7
      - uses: docker/setup-buildx-action@v4
      - uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
```

<v-clicks>

- `GITHUB_TOKEN` is created for each run and **expires** when the job
  ends — no personal token to leak.
- What it can do is set by `permissions:` — here: read code, write
  packages.

</v-clicks>

---
---

# Tags without thinking: `metadata-action`

```yaml {1-8|all}
      - id: meta
        uses: docker/metadata-action@v6
        with:
          images: ghcr.io/farikha/lab11-cicd
          tags: |
            type=sha                          # sha-3f9c2e1
            type=semver,pattern={{version}}   # 1.2.0  (from tag v1.2.0)
            type=ref,event=branch             # main
```

| Event | Tags pushed |
|---|---|
| push to `main` | `main`, `sha-3f9c2e1` |
| tag `v1.2.0` | `1.2.0`, `sha-3f9c2e1` |

<div v-click class="mt-2 text-sm opacity-70">
GHCR image names must be <b>lower case</b> — lower-case
<code>github.repository</code> before using it as an image name.
</div>

---
---

# Build, scan, then push

```yaml {1-6|7-12|13-17}
      - uses: docker/build-push-action@v7       # 1. build, keep it local
        with:
          context: .
          load: true
          tags: lab11-app:scan
          cache-from: type=gha
      - uses: aquasecurity/trivy-action@v0.36.0  # 2. scan the local image
        with:
          image-ref: lab11-app:scan
          severity: HIGH,CRITICAL
          ignore-unfixed: true
          exit-code: "1"
      - uses: docker/build-push-action@v7       # 3. push only if the scan passed
        with:
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

---
---

# Why this order?

<v-clicks>

- A vulnerable image is **never pushed** — the gate fails before step 3.
- `cache-from: type=gha` stores Docker layers in the Actions cache: the
  second build is mostly cache hits.
- The push re-uses the same layers — it doesn't really "build twice".
- `labels` add OCI metadata: source repo, commit, created date — `docker
  inspect` shows which commit an image came from.

</v-clicks>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
Your <code>image</code> job fails with
<code>denied: permission_denied: write_package</code>. The login step
was green. What's missing?
</div>

<div v-click class="mt-8 text-lg opacity-70">
<code>permissions: packages: write</code> on that job. Logging in only
proves who you are; the token's permissions decide what you may do.
</div>

---
layout: section
transition: slide-left
---

# Block 5
## Security gates

---
---

# Gates: a check that can say "no"

| Gate | Tool | Fails when |
|---|---|---|
| Lint / format | ruff | Style or obvious bugs |
| Tests | pytest | Behaviour broke |
| Secrets in git | gitleaks | A token or key is in any commit |
| Image CVEs | Trivy | HIGH/CRITICAL with a fix available |
| Dependency CVEs | pip-audit | A pinned package is known-vulnerable |
| IaC misconfig | `trivy config` | Public buckets, open ports… (next week) |

<div v-click class="mt-4 text-sm opacity-70">
A gate is only a gate if it's <b>required</b> by branch protection and
nobody switches it off "to make it pass".
</div>

---
---

# gitleaks and pip-audit

```yaml {1-8|9-12}
  secrets-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with: { fetch-depth: 0 }      # the whole history, not just HEAD
      - run: |
          docker run --rm -v "$PWD:/repo" zricethezav/gitleaks:v8.30.1 \
            detect --source /repo --redact --exit-code 1
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - run: pipx run pip-audit -r requirements.txt
```

<div v-click class="mt-2 text-sm opacity-70">
A secret removed in a later commit is still in history — that's why
gitleaks scans all commits. Found one? <b>Rotate it first</b>, then clean
history.
</div>

---
---

# Secrets, variables, environments

| | Visible in logs? | Example | Where |
|---|---|---|---|
| **Secret** | Masked `***` | SSH key, API token | Settings → Secrets |
| **Variable** | Yes | `APP_PORT=8080` | Settings → Variables |
| **Environment secret** | Masked | Prod SSH key | Only jobs with `environment: production` |

<v-clicks>

- `${{ secrets.NAME }}` — not available to PRs from **forks** (on
  purpose).
- Never `echo` a secret, never write it to a file that becomes an
  artifact.
- Masking is a safety net, not a guarantee: a secret printed base64 is
  not masked.

</v-clicks>

---
---

# Least privilege for `GITHUB_TOKEN`

```yaml
permissions:
  contents: read          # workflow default: read the code, nothing else

jobs:
  image:
    permissions:
      contents: read
      packages: write     # only the job that pushes
```

<v-clicks>

- With no `permissions:` the token may get broad write access (depends
  on repo settings).
- A compromised step can only do what the token allows.
- Declare permissions per job: each job gets exactly what it needs.

</v-clicks>

---
---

# OIDC: no stored cloud keys at all

<v-clicks>

- The classic way: put a long-lived cloud key in `secrets.` — it can
  leak, and it never expires.
- **OIDC**: the job asks GitHub for a short-lived **identity token**
  (`permissions: id-token: write`), the cloud provider trusts GitHub and
  swaps it for temporary credentials.
- The trust rule can say "only this repo, only branch `main`, only
  environment `production`".
- Every major cloud and tools like Vault support it — you'll see it again
  in your final project.

</v-clicks>

---
layout: section
transition: slide-left
---

# Block 6
## Protect `main`, then deploy

---
---

# Branch protection (rulesets)

<v-clicks>

- **Settings → Rules → Rulesets** (or *Branches* in the older UI) on
  `main`:
- Require a **pull request** before merging (and N approvals in a team).
- Require **status checks**: `lint`, `test (3.11)`, `test (3.12)`,
  `secrets-scan`.
- Block force pushes and deletions.
- Result: a red check = the **Merge** button is disabled. Nobody pushes
  straight to `main`, not even you.

</v-clicks>

---
---

# Environments and approvals

```yaml {3-4}
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production     # needs a reviewer, unlocks prod secrets
```

<v-clicks>

- **Settings → Environments → production → Required reviewers**.
- The job **pauses** until a reviewer clicks *Approve* — that's
  continuous **delivery** with a human gate.
- Environment secrets (prod SSH key) are only given to jobs that name
  that environment.
- The environment page shows **which commit is deployed** and when.

</v-clicks>

---
---

# Deploy over SSH

```yaml {1-6|7-12|13-17}
on:
  workflow_dispatch:
    inputs:
      tag: { description: "Image tag to deploy", required: true }
concurrency: { group: deploy-production, cancel-in-progress: false }
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: appleboy/ssh-action@v1.2.5
        env: { TAG: "${{ inputs.tag }}" }
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          username: ${{ secrets.DEPLOY_USER }}
          key: ${{ secrets.DEPLOY_SSH_KEY }}
          envs: TAG
```

---
---

# …and what runs on the server

```bash
set -euo pipefail
cd ~/lab11
sed -i "s/^TAG=.*/TAG=${TAG}/" .env
docker compose pull
docker compose up -d
docker compose ps
```

<v-clicks>

- Exactly last week's manual release — now in git, reviewed, logged.
- `concurrency` with `cancel-in-progress: false`: two deploys never run
  at once, and a running deploy is never cut in half.
- A dedicated **deploy key**, only for this server, only in the
  environment secrets.

</v-clicks>

---
---

# Rollback = deploy an older tag

<v-clicks>

- Every release is an immutable image in the registry: `1.0.0`, `1.0.1`,
  `1.1.0`…
- Rollback: **Run workflow** with `1.0.0` → approve → done in seconds.
- Nothing is rebuilt — you deploy the exact image that ran before.
- Database migrations are the hard part: make them backwards-compatible
  so the old version still works with the new schema.

</v-clicks>

<div v-click class="mt-6 p-4 rounded bg-blue-500/10 text-sm">
Practise the rollback <b>before</b> you need it. An untested rollback is
just a hope.
</div>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
v1.1.0 is broken in production. Do you revert the commit, wait for the
pipeline and deploy v1.1.1 — or deploy 1.0.1 right now?
</div>

<div v-click class="mt-8 text-lg opacity-70">
Roll back first (deploy 1.0.1: seconds), then fix forward calmly with a
reviewed PR and a new tag. Restore service, then investigate.
</div>

---
layout: section
transition: slide-left
---

# Block 7
## Reuse, runners, debugging

---
---

# Reusable workflows

```yaml
# .github/workflows/python-ci.yml  (called by others)
on:
  workflow_call:
    inputs:
      python-version: { type: string, default: "3.12" }
```

```yaml
# .github/workflows/ci.yml  (the caller)
jobs:
  ci:
    uses: my-org/pipelines/.github/workflows/python-ci.yml@v1
    with: { python-version: "3.12" }
```

<div v-click class="mt-2 text-sm opacity-70">
One pipeline definition, many repositories: fix it once, every team
gets the fix.
</div>

---
---

# Composite actions

```yaml
# .github/actions/setup-app/action.yml
name: setup-app
runs:
  using: composite
  steps:
    - uses: actions/setup-python@v7
      with: { python-version: "3.12", cache: pip }
    - run: pip install -r requirements-dev.txt
      shell: bash
```

```yaml
      - uses: ./.github/actions/setup-app     # in any job of this repo
```

<div v-click class="mt-2 text-sm opacity-70">
Reusable workflow = a whole job (or jobs). Composite action = a few
steps you repeat inside jobs.
</div>

---
---

# Self-hosted runners

<v-clicks>

- **When:** you need special hardware, access to a private network (a
  server behind a firewall), or more minutes than you have.
- **How:** Settings → Actions → Runners → New, run the agent;
  `runs-on: self-hosted`.
- **Risk:** the job's code runs **on your machine**. On a public repo
  anyone can open a PR → never run PR workflows on self-hosted runners.
- Prefer ephemeral runners (a fresh VM/container per job) and the
  minimum access on that machine.

</v-clicks>

---
---

# Debugging a red pipeline

<v-clicks>

- Read the **first** red step's log — later failures are usually
  consequences.
- **Re-run jobs → Enable debug logging** (or secret `ACTIONS_STEP_DEBUG`
  = `true`) for verbose logs.
- Run the same commands locally — the step is "just" a shell command.
- **actionlint** catches YAML and expression mistakes before you push.
- **act** runs workflows locally in Docker — handy, but not identical to
  GitHub's runners.

</v-clicks>

---
---

# When it doesn't work

| Symptom | Likely cause |
|---|---|
| Workflow never starts | Wrong folder, YAML error, or the event filter doesn't match |
| `write_package` denied | Missing `permissions: packages: write` |
| `repository name must be lowercase` | Capitals in user/repo used as image name |
| Check not offered in the ruleset | It hasn't run yet, or the job name differs |
| Secret is empty | PR from a fork, or the job lacks `environment:` |
| Job skipped | A job in `needs:` failed, or `if:` was false |
| Deploy hangs | Waiting for **your** approval in the environment |

---
layout: section
transition: slide-left
---

# Block 8
## Today's practice

---
---

# Today's practice — let the pipeline ship it

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

1. Repo `lab11-cicd` with the app, tests and Dockerfile
2. CI on every PR: lint, tests on 3.11 + 3.12, gitleaks
3. Break a test → red PR → fix → green
4. Ruleset on `main`: checks required, no direct pushes
5. Merge → image built, **Trivy-gated**, pushed to GHCR
6. Tag `v1.0.0` → image tag `1.0.0`

</div>
<div>

```text
PR #1  ci
  ✓ lint
  ✗ test (3.11)   test_greet
  ✗ test (3.12)   test_greet
  ✓ secrets-scan
  – image (skipped)
```

<div v-click class="mt-4 opacity-70">
Stretch: a <code>deploy</code> workflow with a
<code>production</code> approval, then a rollback to the previous tag.
</div>

</div>
</div>

---
---

# By the end of this lesson, you should be able to

<v-clicks>

- [ ] Explain CI, continuous delivery and continuous deployment
- [ ] Read and write a workflow: events, jobs, steps, matrix, `needs`, `if`
- [ ] Build, scan and push an image to GHCR with meaningful tags
- [ ] Add security gates and keep tokens least-privilege
- [ ] Make branch protection enforce your checks
- [ ] Deploy with an approval and roll back to an older tag

</v-clicks>

---
layout: default
---

# Before next lecture

- [ ] Finish today's practice if you didn't wrap it up in session
- [ ] Add the same CI to your final project repository
      (`css4003-final-project` — it already has a minimal `ci.yml`)
- [ ] Install Terraform (`terraform -version`)

<div class="mt-8 text-sm opacity-60">
Next week the infrastructure itself becomes code, reviewed and applied
the same way.
</div>

---
layout: end
---

# Next lecture

Infrastructure as Code with Terraform — describe servers, networks and
storage in code, review them in a PR, and apply them from a pipeline.
