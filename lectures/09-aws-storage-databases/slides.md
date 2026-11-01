---
theme: seriph
title: "CSS4003 — Lecture 9: Cloud Storage & Docker Fundamentals"
info: |
  CSS4003 — DevOps 2 · Narxoz University
  Lecture 9 of 15
background: /cover-bg.svg
transition: fade
mdc: true
download: true
---

# CSS4003 — DevOps 2

## Lecture 9: Cloud Storage & Docker Fundamentals

<div class="pt-8 opacity-70">
Nauruzbayeva Farikha · Lesson 9
</div>

---
layout: default
---

# Recap — Lesson 8 (AWS Networking)

<v-clicks>

- What makes a subnet "public"? <span v-click class="opacity-60">(its route table sends `0.0.0.0/0` to an internet gateway — not its name, not its CIDR)</span>
- How many addresses does a `/24` subnet have? <span v-click class="opacity-60">(256, of which AWS reserves 5 — so 251 usable)</span>
- Why does SG-app allow SSH from SG-web instead of an IP? <span v-click class="opacity-60">(it follows the web servers wherever they move; an IP rule breaks when an instance is replaced)</span>
- Security group vs NACL: which one is stateful? <span v-click class="opacity-60">(the security group — return traffic is allowed automatically)</span>

</v-clicks>

---
---

# Today's agenda

<v-clicks>

- [ ] Part 1 — cloud storage in 15 minutes: EBS, S3, EFS, RDS, DynamoDB
- [ ] Scripting S3: a backup script, boto3, and where credentials come from
- [ ] Part 2 — why containers, and how they differ from VMs
- [ ] Images, containers, registries and the everyday `docker` commands
- [ ] Writing a Dockerfile, and making the layer cache work for you
- [ ] Volumes, networks and Docker Compose
- [ ] Storage meets containers: a container that backs up to S3
- [ ] Live demo → straight into today's practice

</v-clicks>

---
layout: center
class: text-center
---

# The scenario

<div class="text-lg text-left mt-4 max-w-2xl mx-auto">

Your teammate's Flask app "works on my machine". On the server it
crashes: a different Python version, a missing library. When it finally
runs, the notes users save live on the server's disk — until someone
redeploys and they're gone.

</div>

<div v-click class="mt-8 text-xl font-bold">
Today: package the app so it runs the same everywhere — and keep its
data somewhere that survives.
</div>

---
layout: section
transition: slide-left
---

# Part 1
## Cloud storage in 15 minutes

---
---

# Block, file, object

| | Block | File | Object |
|---|---|---|---|
| Looks like | A raw disk | A shared folder | A key → a blob of bytes |
| You use it via | A filesystem you format | NFS mount | HTTP API (`GET`, `PUT`) |
| Attached to | One instance | Many instances | Nothing — reach it from anywhere |
| AWS service | **EBS** | **EFS** | **S3** |
| Good for | OS disk, databases | Shared app files | Uploads, backups, logs, static sites |

<div v-click class="mt-6 text-sm opacity-70">
The scenario's mistake: user data lived only on the server's own disk,
with no copy anywhere else.
</div>

---
---

# EBS vs S3 vs EFS

| | EBS | S3 | EFS |
|---|---|---|---|
| Scope | One AZ, one instance at a time | Region, reached over HTTPS | Region, many instances |
| Survives instance stop | Yes | — | — |
| Backups | Snapshots (incremental) | Versioning, replication | AWS Backup |
| Typical DevOps use | Root disk, DB data files | Artifacts, backups, static sites | Shared uploads folder |

<v-clicks>

- EBS volume types differ in speed and price: **gp3** is the general default.
- **Instance store** is a fast local disk that is **lost on stop** — scratch data only.

</v-clicks>

---
---

# S3 essentials: versioning and lifecycle

<v-clicks>

- **Versioning on** → an overwrite keeps the old copy; `rm` adds a
  **delete marker** instead of erasing. Remove the marker → the file is back.
- Every version is billed, so pair it with a **lifecycle rule** that
  expires old versions and moves old data to cheaper classes.

</v-clicks>

```bash
aws s3api put-bucket-versioning --bucket lab09-123 \
  --versioning-configuration Status=Enabled
aws s3api put-bucket-lifecycle-configuration --bucket lab09-123 \
  --lifecycle-configuration file://lifecycle.json   # rules live in Git
```

---
---

# S3 essentials: keep it private, share briefly

<v-clicks>

- New buckets are **private**. **Block Public Access** (all four switches on)
  stops any policy or ACL from making them public by mistake.
- To share **one** file, hand out a **presigned URL**: a link with a
  signature and an expiry, using **your** permissions.

</v-clicks>

```bash
aws s3api put-public-access-block --bucket lab09-123 \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
aws s3 presign s3://lab09-123/report.pdf --expires-in 300   # valid 5 minutes
```

---
---

# S3 from the CLI — what scripts and CI/CD use

```bash {1|2|3|4|5}
aws s3 ls s3://lab09-123/                            # list a prefix
aws s3 cp build.zip s3://lab09-123/releases/v1.zip   # upload one file
aws s3 cp ./data s3://lab09-123/backup/ --recursive  # upload a folder
aws s3 sync ./dist s3://lab09-123/site/ --delete     # mirror (careful!)
aws s3api get-bucket-versioning --bucket lab09-123   # low-level API
```

<v-clicks>

- **`aws s3`** — file-like commands: `ls`, `cp`, `sync`, `rm`.
- **`aws s3api`** — one command per API call: versioning, lifecycle, policies.
- `sync --delete` removes what isn't in your folder: run `--dryrun` first.

</v-clicks>

---
---

# Databases: Amazon RDS

| | DB on EC2 | Amazon RDS |
|---|---|---|
| OS and DB patching | You | AWS |
| Backups | You script them | Built in, with point-in-time restore |
| Standby in another AZ | You build replication | One setting: **Multi-AZ** |
| Your job | Everything | Schema, data, who can connect, cost |

<v-clicks>

- Runs in **private subnets** of your VPC; you connect to an **endpoint** (DNS name).
- **Multi-AZ** = availability (same endpoint). **Read replica** = more reads (own endpoint).

</v-clicks>

---
---

# Databases: DynamoDB

<v-clicks>

- Fully managed **key-value / document** database: no servers, no patching.
- Every item has a **partition key** (e.g. `id`); other attributes are free-form.
- **On-demand** capacity: pay per request — good for spiky traffic.

</v-clicks>

```bash
aws dynamodb put-item --table-name students \
  --item '{"id":{"S":"s001"},"name":{"S":"Aigerim"}}'
aws dynamodb get-item --table-name students --key '{"id":{"S":"s001"}}'
```

---
---

# Which one for which data?

| Data | Service | Why |
|---|---|---|
| OS disk of an instance | EBS (gp3) | Block disk, survives stop/start |
| Uploads, backups, logs, artifacts | S3 (+ lifecycle) | Durable, cheap, reachable from anywhere |
| Files shared by several servers | EFS | One file system, many instances |
| Relational app data | RDS (Multi-AZ in production) | SQL, joins, managed backups |
| Simple key lookups at scale | DynamoDB | Serverless, pay per request |

---
---

# `backup.sh` (1/2): fail fast, name things

```bash {1-4|6-7|8-10}
#!/usr/bin/env bash
# Back up a folder to S3 under backups/<today>/
# usage: ./backup.sh <folder> <bucket>
set -euo pipefail

SRC=${1:?usage: backup.sh <folder> <bucket>}
BUCKET=${2:?usage: backup.sh <folder> <bucket>}
STAMP=$(date +%F-%H%M%S)                  # 2026-11-03-101500
NAME="$(basename "$SRC")-$STAMP.tar.gz"
KEY="backups/$(date +%F)/$NAME"           # one "folder" per day
```

<div v-click class="mt-4 text-sm opacity-70">
<code>-e</code> stops on the first failed command, <code>-u</code> on an unset
variable, <code>-o pipefail</code> when any part of a pipe fails.
<code>${1:?msg}</code> exits with a usage message if the argument is missing.
</div>

---
---

# `backup.sh` (2/2): pack, upload, show

```bash {1|2|3-4}
tar -czf "/tmp/$NAME" -C "$(dirname "$SRC")" "$(basename "$SRC")"
aws s3 cp "/tmp/$NAME" "s3://$BUCKET/$KEY"
echo "Backed up $SRC to s3://$BUCKET/$KEY"
aws s3 ls "s3://$BUCKET/backups/" --recursive
```

```text
$ ./backup.sh myapp lab09-123
upload: /tmp/myapp-2026-11-03-101500.tar.gz to s3://lab09-123/backups/2026-11-03/...
Backed up myapp to s3://lab09-123/backups/2026-11-03/myapp-2026-11-03-101500.tar.gz
```

<div v-click class="mt-4 text-sm opacity-70">
Thanks to <code>set -e</code>, a failed upload stops the script — it never
prints "Backed up" for a backup that doesn't exist. Later today the same
idea runs inside a container.
</div>

---
---

# Python + boto3: S3

```python {1-2|4|5|6-8}
import boto3
s3 = boto3.client("s3")

s3.upload_file("notes.txt", "lab09-123", "docs/notes.txt")
s3.download_file("lab09-123", "docs/notes.txt", "copy.txt")
url = s3.generate_presigned_url("get_object",
        Params={"Bucket": "lab09-123", "Key": "docs/notes.txt"},
        ExpiresIn=300)                 # seconds
```

<div v-click class="mt-4 text-sm opacity-70">
Upload, download, presigned URL — the CLI commands, but inside a program
that can loop, check results and decide.
</div>

---
---

# Credentials: never in the code

<v-clicks>

- The AWS CLI and boto3 look for credentials in a fixed order:
  **environment variables** → **`~/.aws/credentials`** → **instance role** on EC2.
- So the same script runs on your laptop, in Learner Lab and on EC2 — with
  **zero keys in the file**. Region: `AWS_DEFAULT_REGION=us-east-1`.
- `AWS_ENDPOINT_URL=http://localhost:5050` sends every call to **moto**, a
  local S3, instead of AWS. Remember this one — containers use it today.

</v-clicks>

<div v-click class="mt-4 p-4 rounded bg-red-500/10 text-sm">
<code>aws_access_key_id="AKIA…"</code> in a file that reaches GitHub is a
leaked key within minutes. Bots scan public repos for them.
</div>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
Versioning is on. You run <code>aws s3 rm s3://bucket/notes.txt</code>.
Is the file gone?
</div>

<div v-click class="mt-8 text-lg opacity-70">
No. S3 adds a <b>delete marker</b> as the newest version. The older versions
are still there — delete the marker and the file is back.
</div>

---
layout: section
transition: slide-left
---

# Part 2
## Docker fundamentals

---
---

# "It works on my machine"

<v-clicks>

- An app needs more than its code: a **language runtime**, **libraries**
  with exact versions, OS packages, config.
- Every laptop and server has a slightly different mix → "works here,
  crashes there".
- A **container** ships the app **together with** everything it needs, as
  one package that runs the same on any machine with Docker.
- For DevOps this means: **build once, run anywhere** — laptop, CI, EC2, Kubernetes.

</v-clicks>

---
---

# Containers vs virtual machines

| | Virtual machine | Container |
|---|---|---|
| Contains | A whole OS + kernel + app | App + its libraries, shares the host kernel |
| Size | Gigabytes | Often tens of megabytes |
| Starts in | Minutes | Seconds or less |
| Isolation | Strong (hypervisor) | Process-level (namespaces, cgroups) |
| On AWS | EC2 instance | Runs **on** an EC2 instance (or ECS, EKS) |

<div v-click class="mt-6 text-sm opacity-70">
Containers don't replace VMs: in the cloud you usually run containers
<b>on</b> VMs. The container makes the app portable; the VM provides the machine.
</div>

---
---

# Image vs container

<v-clicks>

- An **image** is a read-only template: a filesystem snapshot + metadata
  (which command to start). Built once, stored, shared.
- A **container** is a **running instance** of an image, with its own thin
  writable layer on top.
- One image → many containers. Delete a container and its writable layer
  is gone; the image stays.
- Analogy: image = class, container = object. Or: image = `.iso`, container
  = the running machine.

</v-clicks>

---
---

# Registries and tags

<v-clicks>

- A **registry** stores images: **Docker Hub** (public default), GitHub
  Container Registry, AWS **ECR**.
- Image names: `[registry/]repository:tag` — `nginx:1.31-alpine`,
  `python:3.12-slim`, `ghcr.io/org/app:2.0`.
- No tag means `:latest` — which changes under your feet. **Pin a version**
  in anything you deploy.
- `alpine` / `slim` variants are smaller base images: less to download,
  fewer packages to patch.

</v-clicks>

---
---

# Your first container

```bash {1|2|3|4}
docker run -d --name web -p 8080:80 nginx:1.31-alpine
docker ps                           # running containers
curl -s localhost:8080 | grep title # <title>Welcome to nginx!</title>
docker ps -a                        # also the stopped ones
```

<v-clicks>

- `run` = pull the image if missing, create a container, start it.
- `-d` — detached, in the background. `--name` — so you don't copy IDs.
- `-p 8080:80` — **host port 8080 → container port 80**. Without `-p`,
  nothing outside the container can reach it.

</v-clicks>

---
---

# Inspect, get inside, stop

```bash {1|2|3|4|5}
docker logs web               # stdout/stderr of the main process
docker logs -f web            # follow, like tail -f
docker exec -it web sh        # a shell INSIDE the running container
docker stop web               # graceful stop (SIGTERM, then SIGKILL)
docker rm web                 # delete it (or: docker rm -f web)
```

<v-clicks>

- Containers should log to **stdout**, not to files — `docker logs` (and
  later CloudWatch) collects it.
- `exec -it` is for debugging. Never "fix" a container by hand: rebuild the image.

</v-clicks>

---
---

# Working with images

```bash {1|2|3|4}
docker pull python:3.12-slim     # download from the registry
docker images                    # what's stored locally, with sizes
docker history python:3.12-slim  # the layers an image is made of
docker rmi lab09-app:slow        # delete a local image
```

<div v-click class="mt-6 text-sm opacity-70">
Images are cached locally after the first pull — the second
<code>docker run</code> of the same tag starts instantly.
</div>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
You ran <code>docker run -d nginx:1.31-alpine</code> and <code>curl
localhost:80</code> fails. The container is running. Why?
</div>

<div v-click class="mt-8 text-lg opacity-70">
No <code>-p</code>: port 80 exists only inside the container's network.
Publish it with <code>-p 8080:80</code> and curl <code>localhost:8080</code>.
</div>

---
layout: section
transition: slide-left
---

# Building images
## The Dockerfile

---
---

# Dockerfile instructions you need today

| Instruction | What it does |
|---|---|
| `FROM python:3.12-slim` | Base image to start from |
| `WORKDIR /app` | Working directory for the next steps (created if missing) |
| `COPY src dest` | Copy files from the build folder into the image |
| `RUN pip install …` | Run a command **at build time**, save the result as a layer |
| `ENV KEY=value` | Environment variable for build and run |
| `EXPOSE 8000` | Documents the port — does **not** publish it |
| `CMD ["python", "app.py"]` | Default command when a container **starts** |

---
---

# The app we'll package

```python {1-5|7-10|12-17|19-20}
import os, socket
from flask import Flask

app = Flask(__name__)
DATA = os.environ.get("DATA_DIR", "/data")

@app.get("/")
def index():
    os.makedirs(DATA, exist_ok=True)
    return {"host": socket.gethostname(), "files": sorted(os.listdir(DATA))}

@app.post("/notes/<name>")
def add_note(name):
    os.makedirs(DATA, exist_ok=True)
    with open(os.path.join(DATA, f"{name}.txt"), "w") as f:
        f.write(f"note {name}\n")
    return {"saved": f"{name}.txt"}, 201

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)
```

---
---

# The Dockerfile

```dockerfile {1|2|3-4|5|6-7|8}
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
ENV DATA_DIR=/data
EXPOSE 8000
CMD ["python", "app.py"]
```

<v-clicks>

- `requirements.txt` contains one line: `flask==3.1.3` — pinned.
- The app binds **`0.0.0.0`**: inside a container, `127.0.0.1` would only
  accept connections from the container itself.

</v-clicks>

---
---

# Build and run

```bash {1|2|3|4|5}
docker build -t lab09-app:1.0 .
docker run -d --name app -p 8000:8000 lab09-app:1.0
curl -s localhost:8000/                      # {"files":[],"host":"32d95490db47"}
curl -s -X POST localhost:8000/notes/first   # {"saved":"first.txt"}
curl -s localhost:8000/                      # {"files":["first.txt"],"host":"32d95490db47"}
```

<v-clicks>

- `-t name:tag` names the image; `.` is the **build context** sent to Docker.
- `host` is the **container's** hostname — its short ID, not your laptop.

</v-clicks>

---
---

# Layers and the build cache

<v-clicks>

- Each `FROM`, `COPY` and `RUN` produces a **layer**. Layers are cached.
- On a rebuild Docker reuses every layer **until the first step whose input
  changed** — that step and everything after it run again.
- `COPY` changes when the copied files change; `RUN` when the step above it changed.
- So: put what changes **rarely** (dependencies) **first**, what changes
  **often** (your code) **last**.

</v-clicks>

```text
#8 [4/5] RUN pip install --no-cache-dir -r requirements.txt
#8 CACHED
#9 [5/5] COPY app.py .
```

---
---

# Order matters

<div class="grid grid-cols-2 gap-6">
<div>

**Slow** — any code change reinstalls everything

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["python", "app.py"]
```

</div>
<div>

**Fast** — dependencies stay cached

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app.py .
CMD ["python", "app.py"]
```

</div>
</div>

<div v-click class="mt-6 text-sm opacity-70">
In CI this is the difference between a 10-second and a 3-minute build —
on every push.
</div>

---
---

# `.dockerignore`

```text
.venv/
__pycache__/
.git/
.env
*.tar.gz
```

<v-clicks>

- Everything in the build folder is sent to Docker as the **build context**.
- `.dockerignore` keeps out junk (faster builds, smaller images) and
  **secrets** — a `.env` copied by `COPY . .` would end up inside the image.

</v-clicks>

---
---

# `CMD` vs `ENTRYPOINT`

<v-clicks>

- **`CMD`** — the default command. Replaced by anything you type after the
  image name: `docker run lab09-app:1.0 python -V` runs `python -V` instead.
- **`ENTRYPOINT`** — the fixed program; `CMD` / your arguments become its
  **arguments**. Used by tool images: `amazon/aws-cli` has entrypoint `aws`,
  so `docker run amazon/aws-cli:2.39.3 s3 ls` runs `aws s3 ls`.
- To run something else in such an image, override it: `--entrypoint sh`.
- Use the JSON form `["python", "app.py"]`: the app gets signals
  (`docker stop`) directly, without a shell in between.

</v-clicks>

---
---

# Tag and push (concept)

```bash {1|2|3|4}
docker tag lab09-app:1.0 myuser/lab09-app:1.0   # name it for Docker Hub
docker login                                     # once per machine
docker push myuser/lab09-app:1.0                 # upload the layers
docker pull myuser/lab09-app:1.0                 # on any other machine
```

<v-clicks>

- The server never builds anything: it **pulls** the exact image you tested.
- Next week: the same with **AWS ECR**, and running the image on EC2.

</v-clicks>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
You changed one line in <code>app.py</code> and rebuilt. With the "fast"
Dockerfile, which steps run again?
</div>

<div v-click class="mt-8 text-lg opacity-70">
Only <code>COPY app.py .</code> and the steps after it. <code>FROM</code>,
<code>COPY requirements.txt</code> and <code>pip install</code> come from the
cache, because their inputs didn't change.
</div>

---
layout: section
transition: slide-left
---

# Data, networks, Compose
## Containers that keep data and talk to each other

---
---

# Containers are disposable — data isn't

<v-clicks>

- A container's writable layer is deleted with the container.
  Redeploy = new container = **empty** again (today's scenario).
- A **volume** is storage managed by Docker, mounted into the container
  at a path. It outlives any container.

</v-clicks>

```bash {1|2|3-4}
docker volume create lab09-data
docker run -d --name app -p 8000:8000 -v lab09-data:/data lab09-app:1.0
docker rm -f app        # container gone…
docker run -d --name app -p 8000:8000 -v lab09-data:/data lab09-app:1.0   # …files still there
```

---
---

# Named volume vs bind mount

| | Named volume | Bind mount |
|---|---|---|
| Syntax | `-v lab09-data:/data` | `-v "$PWD/site":/usr/share/nginx/html` |
| Lives in | Docker's storage area | A folder **you** choose on the host |
| Created by | Docker (automatically if missing) | You |
| Good for | Databases, app data | Code and config during development |

<div v-click class="mt-6 text-sm opacity-70">
A typo in a volume name doesn't fail — Docker silently creates a new,
empty volume. Check with <code>docker volume ls</code>.
</div>

---
---

# Networks: containers find each other by name

<v-clicks>

- Containers on the same **user-defined network** reach each other by
  **container (or service) name** — Docker runs a DNS for them.
- No published ports needed between them: `-p` is only for traffic from
  **outside**.
- Never hardcode container IPs: they change on every restart.

</v-clicks>

```bash
docker network create labnet
docker run -d --name s3 --network labnet motoserver/moto:5.2.5
docker run --rm --network labnet python:3.12-slim getent hosts s3   # <its IP>  s3
```

---
---

# Docker Compose: the whole stack in one file

<v-clicks>

- Real apps are several containers: the app, a database, a cache, a
  local S3…
- **Compose** describes them in `compose.yaml`: images or builds, ports,
  volumes, env — and puts them on **one network** automatically.
- One command starts everything; the file lives in Git next to the code.

</v-clicks>

---
---

# `compose.yaml`: app + local S3

```yaml {1-7|9-12|14-15}
services:
  app:
    build: .
    image: lab09-app:1.0
    ports: ["8000:8000"]
    volumes:
      - appdata:/data

  s3:
    image: motoserver/moto:5.2.5
    ports:
      - "5050:5000"   # host 5050 (5000 is AirPlay on macOS)

volumes:
  appdata:
```

<div v-click class="mt-2 text-sm opacity-70">
Inside the stack, the app reaches the S3 service at <code>http://s3:5000</code>
— service name + container port.
</div>

---
---

# Compose commands

```bash {1|2|3|4|5|6-7}
docker compose up -d --build        # build if needed, start everything
docker compose ps                   # status of the services
docker compose logs -f app          # logs of one service
docker compose exec app sh          # shell in a running service
docker compose run --rm backup      # start a one-off job, delete it after
docker compose down                 # stop + remove containers and network
docker compose down -v              # …and the volumes (= your data!)
```

---
---

# Storage meets containers

```yaml {1-4|5-6|7-12}
  backup:
    image: amazon/aws-cli:2.39.3
    profiles: ["tools"]          # not started by "up", only by "run"
    env_file: .env               # keys + AWS_ENDPOINT_URL — not in the image
    volumes:
      - appdata:/data:ro         # the app's data, read-only
    entrypoint: ["/bin/sh", "-c"]
    command:
      - |
        aws s3 mb "s3://$$BUCKET" 2>/dev/null || true
        aws s3 cp /data "s3://$$BUCKET/backups/$$(date +%F)/" --recursive
```

<div v-click class="mt-2 text-sm opacity-70">
With <code>AWS_ENDPOINT_URL=http://s3:5000</code> in <code>.env</code> it backs up
to the local moto container; remove that line and put Learner Lab keys there —
the same container backs up to real S3.
</div>

---
---

# Secrets: never baked into an image

<v-clicks>

- Anything you `COPY` or `ENV` into an image is readable by everyone who
  can pull it: `docker history`, or just `docker run … env`.
- Pass secrets **at run time**: `-e`, `--env-file .env`, Compose
  `env_file:` — and keep `.env` in `.gitignore` **and** `.dockerignore`.
- In AWS, containers get credentials from an **IAM role** (instance or task
  role) — no keys at all. Same idea as last week's credentials chain.

</v-clicks>

<div v-click class="mt-4 p-4 rounded bg-red-500/10 text-sm">
A pushed image with an access key inside is a public leak — deleting the tag
doesn't help, the image may already be pulled. Rotate the key.
</div>

---
---

# Cleaning up

```bash {1|2|3|4}
docker ps -a                  # stopped containers pile up
docker system df              # how much space images, containers, volumes use
docker image prune            # remove dangling (untagged) images
docker system prune           # stopped containers, unused networks, dangling images
```

<div v-click class="mt-6 p-4 rounded bg-red-500/10 text-sm">
<code>docker system prune -a --volumes</code> also deletes <b>all</b> unused images
and <b>volumes</b> — your databases' data included. Read the prompt before
typing <code>y</code>.
</div>

---
---

# When it doesn't work

| Symptom | Likely cause |
|---|---|
| `port is already allocated` | An old container still uses the port: `docker ps -a` |
| `curl` gets no reply | App binds `127.0.0.1` inside the container, or it exited: `docker logs` |
| Data gone after redeploy | No volume, or a typo created a new empty one |
| `pip install` on every build | Code copied before requirements |
| Container can't reach `localhost:5050` | Inside a container, use the service name: `http://s3:5000` |

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
You run <code>docker compose down -v</code> at the end of the day. Tomorrow the
app starts with no notes. Bug?
</div>

<div v-click class="mt-8 text-lg opacity-70">
No — <code>-v</code> deleted the <code>appdata</code> volume. Plain
<code>docker compose down</code> keeps volumes. (And that's why the backup to
S3 exists.)
</div>

---
---

# Try it yourself

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

**Part A — Docker basics** (everyone)

1. `nginx` with `-p`, `logs`, `exec`
2. Write the Dockerfile for the Flask app, build, run
3. Edit `app.py`, rebuild: see `CACHED`
4. Named volume: data survives `docker rm`

**Part B — Compose + S3**

5. `compose.yaml` with `app` + `s3`
6. One-off `backup` container → S3

</div>
<div>

**Track B** — local S3 (moto container):

```bash
AWS_ENDPOINT_URL=http://s3:5000
docker compose run --rm backup
```

**Track A** — real S3 in **Learner Lab**
(`us-east-1`): same compose file, Learner Lab keys
in `.env`, no endpoint line. Delete the bucket and
**End Lab** after.

</div>
</div>

---
---

# By the end of this lesson, you should be able to

<v-clicks>

- [ ] Choose between EBS, S3, EFS, RDS and DynamoDB for a given kind of data
- [ ] Script S3 with the CLI and boto3 without putting keys in the code
- [ ] Explain images vs containers, and containers vs VMs
- [ ] Run, inspect, debug and remove containers with the `docker` CLI
- [ ] Write a Dockerfile that uses the layer cache well
- [ ] Keep data in volumes and connect services with Compose
- [ ] Back up a container's data to S3 with credentials passed at run time

</v-clicks>

---
layout: default
---

# Before next lecture

- [ ] Finish the practice and clean up: `docker compose down -v`, remove
      the lab images and volume
- [ ] Track A: delete your bucket and **End Lab**
- [ ] Create a free **Docker Hub** account if you don't have one

<div class="mt-8 text-sm opacity-60">
Next week we take today's image to AWS: push it to a registry and run it on
EC2.
</div>

---
layout: end
---

# Next lecture

Docker + AWS — pushing images to a registry and running containers on EC2.
