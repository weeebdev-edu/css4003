# CSS4003 · Final Project Report — <Your name>

> Export this file to **PDF** and submit it. Replace every `<…>`. Every section needs
> **evidence**: a screenshot, command output (as text), or a link to a file, commit,
> PR or pipeline run in your repository. No evidence = no points (see `RUBRIC.md`).

| | |
|---|---|
| Student | <Name Surname>, <student ID> |
| Repository | <https://github.com/…> |
| Final commit / tag | <`final` → commit SHA> |
| Track | AWS (own account / Learner Lab) / GCP |
| Demo video | <link> |
| Live URL (if still running) | <url> |

## 0. My variant

| Field | My value | Where it is applied (file / screenshot) |
|---|---|---|
| `theme` | | |
| `resource_prefix` | | |
| `vpc_cidr` | | |
| `db_engine` | | |
| `clickhouse_required` | | |
| `cluster` | | |
| `logging_stack` | | |
| `password_min_length` | | |
| `password_require_symbol` | | |
| `hpa_cpu_target` | | |
| `alert_error_rate_pct` | | |
| `alert_p95_ms` | | |
| `backup_cron_utc` | | |
| `backup_retention_days` | | |

## 1. Architecture
<Diagram (image or Mermaid): users → ingress → app pods → DB; registry, secret manager, object storage, monitoring, logging, CI/CD. Mark public vs private.>

<3–5 sentences: what runs where and why.>

## 2. Linux operations
- Scripts: <links to `ops/backup.sh`, `ops/restore.sh`, `ops/healthcheck.sh`>
- Schedule (`backup_cron_utc`): <CronJob / timer link + recent runs>
- VM bootstrap: <user data / startup script link>
- Evidence: <a failing and a passing healthcheck>

## 3. Git workflow
- Branch protection: <screenshot of the rule>
- PRs: <link to closed PRs; count>
- Releases: <links to ≥ 3 Releases + CHANGELOG>
- gitleaks on full history: <output>

## 4. Cloud & network
- VPC / subnets with `vpc_cidr`: <`terraform output` or console>
- Firewall rules: <table: rule, source, port, why>
- DB not public: <failed connection from your laptop>
- No keys on VMs: <identity check on a node showing the role/service account>

## 5. Docker & registry
- Dockerfile: <link>; final image size: <…>; runs as: <`id` output>
- Registry: <listing showing SHA + semver tags>

## 6. Orchestration (cluster)
- Cluster (`cluster` variant): <`kubectl get nodes -o wide`>
- Manifests: <link to `k8s/`>
- `kubectl get deploy,svc,ingress,hpa,pvc -A`: <output>
- Probes and limits: <excerpt>
- HPA at `hpa_cpu_target` scaling under load: <before/after>
- Node failure drill (2-node / managed): <cordon/drain or stopped VM → pods rescheduled, app still answering>
- Rolling update with zero errors + rollback: <commands + load-tool output>

## 7. Database: roles, policy, backups
- Engine/version (`db_engine`) and where it runs: <…>
- Roles and grants: <SQL link + proof `app_ro` can't INSERT, `app_rw` can't DROP>
- Password policy in DB (`password_min_length`, `password_require_symbol`): <config + a rejected weak password>
- Policy via `/api/admin/check-password`: <weak and strong examples>
- Backups in object storage: <listing with dates>; retention `backup_retention_days`: <lifecycle rule>
- **Restore test**: <steps + before/after query output>
- ClickHouse (if `clickhouse_required: yes`): <query/panel>

## 8. Secret management
- Where secrets live: <secret manager screenshot, values hidden>
- How they reach pods: <ESO manifests or sync job link>
- Rotation: <what you rotated, proof the app still works>
- Proof nothing is in git/images: <gitleaks + image history>

## 9. Security checks (CI gates)
| Gate | Tool | Workflow file | Blocking? |
|---|---|---|---|
| Secrets | gitleaks | | |
| Image | Trivy | | |
| Dependencies | pip-audit | | |
| IaC | trivy config / tfsec / checkov | | |
| Password policy | | | |

- A PR blocked by a gate and its fix: <links>
- Accepted exceptions (if any) with reasons: <…>

## 10. Monitoring
- Prometheus targets up: <screenshot>
- Grafana dashboard (JSON in repo): <link + screenshot>
- Alert rules (`alert_error_rate_pct`, `alert_p95_ms`, + one more): <link>
- An alert fired on purpose: <how + screenshot>

## 11. Logging
- Stack (`logging_stack`) and retention: <…>
- Query tracing one `request_id`: <query + result>

## 12. CI/CD
- Workflows: <links>
- A PR run, a `main` deploy run, a tag release run with approval: <links>
- Rollback: <command/workflow + proof>
- Cloud auth from CI: <OIDC / WIF / self-hosted runner / documented short-lived creds>

## 13. Infrastructure as code
- Terraform layout: <tree>
- Clean `plan` after `apply`: <output>
- Destroy + apply from zero: <outputs or video link>
- Where state lives: <…>

## 14. Decisions (ADRs)
| # | Decision | Alternatives considered | Why |
|---|---|---|---|
| 0001 | | | |
| 0002 | | | |
| 0003 | | | |
| 0004 | | | |

## 15. Cost and teardown
- Monthly estimate (Pricing Calculator link): <…>
- What I destroyed and when: <`terraform destroy` output>

## 16. Reflection
- What was hardest and how I solved it: <…>
- What I would do next with more time: <…>
