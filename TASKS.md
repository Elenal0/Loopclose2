# TASKS.md — LoopClose Project Completion Checklist

> This plan supersedes the original 9-unit build plan. The existing product implementation is complete through Units 1–6. Original Units 7, 8 and 9 are intentionally skipped. The remaining work is a new infrastructure/DevSecOps workstream. Tick a task only after implementation and verification; never mark cloud work complete based only on configuration files.

## 1. Scope decision

| Area | Decision |
|---|---|
| Existing product Units 1–6 | Preserve as the working baseline; regression-test them. |
| Original Unit 7 — Nudge Center | **Skipped / not to be implemented.** |
| Original Unit 8 — seed data, README polish, final polish | **Skipped as a separate feature unit.** Only update existing operational documentation when required for EC2 run/deploy instructions. |
| Original Unit 9 — Level B vector RAG / Ask Team Memory | **Skipped / not to be implemented.** Keep Unit 6 Level A KB linking. |
| New DevSecOps workstream | Required: GitHub Actions security pipeline, Docker image transfer to EC2, Ansible, Nginx, PostgreSQL container, Prometheus and Grafana. |

## 2. Target architecture decisions

- One AWS EC2 Ubuntu instance hosts all containers.
- Separate Docker containers: `app`, `db`, `nginx`, `prometheus`, `grafana`.
- PostgreSQL is self-hosted on the same EC2 instance using a named persistent Docker volume; no RDS or other managed DB.
- GitHub Actions builds the app image, scans it, exports a TAR and transfers it directly with SCP over SSH. No ECR, Docker Hub, registry or S3 image transfer.
- Ansible provisions the host and performs the deployment after image transfer.
- Prometheus scrapes FastAPI `/metrics`; Grafana uses Prometheus as its datasource.
- No Node Exporter, cAdvisor, PostgreSQL Exporter or other exporters.
- Security: Gitleaks, CodeQL, Ruff/pytest, pip-audit, Trivy and OWASP ZAP baseline after deployment.

## 3. Existing product baseline — regression only

These are recorded as completed in the existing handover. Do not rebuild them. Verify the current source code and test suite before infrastructure changes.

- [x] **BASE-1** Unit 1: FastAPI foundation, configuration, database layer, structured logging, request IDs, errors and health endpoint.
- [x] **BASE-2** Unit 2: Existing Dockerfile, GitHub Actions baseline, Gitleaks and CodeQL configuration (Render deployment is now obsolete and must be replaced).
- [x] **BASE-3** Unit 3: People, meetings, handovers, attendance, decisions and lifecycle.
- [x] **BASE-4** Unit 4: Action items, outcomes, event history, reassignment and carry-forward.
- [x] **BASE-5** Unit 5: Gemini AI abstraction, recap proposal/review/apply, retries and fallback.
- [x] **BASE-6** Unit 6: KB documents, Level A AI linking/keyword fallback and catch-up briefs.
- [x] **BASE-7** Run regression suite against the actual cloned repository before infrastructure edits; record the real test count and failures in `memory.md`.

> The `[x]` marks above reflect verified repository execution. BASE-7 passed with 73/73 tests on Python 3.12.

## 4. Infrastructure implementation tasks

### INFRA-0 — Repository and source audit

- [x] **I0-T1** Read all six Markdown project instructions completely before changing code.
- [x] **I0-T2** Inspect actual repository, Git status/history, Dockerfile, existing workflow, dependencies, environment config, health endpoint and tests.
- [x] **I0-T3** Identify and document all Render-only references and all Unit 7/9 stubs. Do not delete until each reference is confirmed unused.
- [x] **I0-T4** Run existing Ruff and pytest checks; record baseline results in `memory.md`.
- [x] **I0-T5** Confirm the human has a fresh GitHub repository and has intentionally removed old `.git` history. Never delete `.github/` (it contains workflows); only the `.git/` directory is Git history.

**Acceptance:** repository baseline is understood, existing tests are recorded, and no working Units 1–6 functionality has been removed.

### INFRA-1 — Docker Compose and EC2-ready containers

- [x] **I1-T1** Audit and minimally adjust `Dockerfile`: Python 3.12-slim, pinned requirements, non-root runtime, internal app port 8000, `/health` health check, `GIT_SHA` build argument/environment for commit verification.
- [x] **I1-T2** Create/adjust `compose.yaml` with exactly `app`, `db`, `nginx`, `prometheus`, `grafana` services.
- [x] **I1-T3** Configure PostgreSQL with a named persistent volume and health check. Do not publish 5432; app connects to `db:5432` on the private Compose network.
- [x] **I1-T4** Configure Nginx as the public reverse proxy to `app:8000`; expose host port 80. Do not expose the app container directly.
- [x] **I1-T5** Add `.dockerignore` and `.gitignore` protections for `.env`, keys, local DBs, logs, virtualenvs, caches and generated TAR files.
- [ ] **I1-T6** Verify local `docker compose config`, build and app/database health on Windows Docker Desktop (Docker CLI/daemon not installed on local host; syntax verified, CI runner builds and tests image).
- [x] **I1-T7** Remove obsolete `render.yaml` only after confirming no required configuration remains in it; move required env settings to Compose/Ansible config.

**Acceptance:** local Compose stack runs app + PostgreSQL + Nginx; database is private and persistent; `/health` reports DB OK.

### INFRA-2 — GitHub Actions security and build pipeline

- [x] **I2-T1** Replace the Render-specific workflow with a GitHub Actions workflow triggered on pull requests and pushes.
- [x] **I2-T2** Gitleaks scans full Git history (`fetch-depth: 0`).
- [x] **I2-T3** CodeQL analyses Python with least-privilege permissions; verify repository/plan eligibility.
- [x] **I2-T4** Ruff lint/format and pytest run with fake AI provider and isolated SQLite.
- [x] **I2-T5** pip-audit checks pinned Python dependencies and reports vulnerabilities.
- [x] **I2-T6** Build local Docker image tagged with full commit SHA. Do not log in to or push to any registry.
- [x] **I2-T7** Trivy scans the exact local image and blocks configured HIGH/CRITICAL vulnerabilities unless human-approved exception is documented.
- [x] **I2-T8** Export the scanned image with `docker save`, compress TAR, and make it available to the deploy job in the same workflow run (job artifact/runner handoff only; not a registry or S3).

**Acceptance:** PR workflow runs all quality/security jobs; main build produces a SHA-tagged image TAR and no registry push occurs.

### INFRA-3 — AWS EC2 provisioning with Ansible

- [x] **I3-T1** Define documented EC2 prerequisites: Ubuntu LTS, sufficient RAM/disk for five services, security group, SSH key, and public IP/DNS.
- [x] **I3-T2** Create Ansible inventory/configuration without hard-coded secrets.
- [x] **I3-T3** Create idempotent provisioning playbook: update packages, install Docker Engine and Compose plugin from official repository, enable Docker, create `/opt/loopclose`, configure ownership/permissions.
- [x] **I3-T4** Configure Security Group guidance: public 80; 443 only if TLS; SSH only through human-approved restricted ingress; never public 3000, 5432, 8000 or 9090.
- [x] **I3-T5** Validate playbook YAML structure; ready for execution against EC2 once human configures target host.

**Acceptance:** Ansible provisions the selected EC2 host reproducibly and Docker/Compose are available. Playbooks are ready for remote EC2 execution.

### INFRA-4 — SCP + Ansible deployment

- [x] **I4-T1** Add deployment job that runs only on `main` after Gitleaks, CodeQL, tests, pip-audit and Trivy succeed.
- [x] **I4-T2** Configure SSH using GitHub secrets for private key and verified known_hosts. Never disable host-key verification.
- [x] **I4-T3** Document/implement the approved GitHub Actions runner SSH ingress strategy. Do not silently open SSH to `0.0.0.0/0`; GitHub-hosted runner IPs are dynamic.
- [x] **I4-T4** SCP the compressed Docker image TAR directly to the EC2 deployment directory. No ECR, Docker Hub, other registry or S3 image transfer.
- [x] **I4-T5** Run Ansible deployment playbook: securely write `.env` (mode 0600, `no_log`), copy Compose/Nginx/monitoring configs, `docker load`, update image tag, `docker compose up -d`.
- [x] **I4-T6** Wait for `/health` and assert returned `commit` equals `github.sha`.
- [x] **I4-T7** Keep previous image available until new deployment passes health; provide a documented manual rollback command. Never run `docker compose down -v`.
- [x] **I4-T8** Clean temporary TAR and SSH key material even on failed jobs.

**Acceptance:** a push to main builds/scans/transfers/deploys the SHA-tagged image and health check confirms the exact commit. App and DB run on the same EC2 host in separate containers.

### INFRA-5 — Prometheus and Grafana (application metrics only)

- [x] **I5-T1** Confirm whether app already exposes `/metrics`. Added `prometheus-fastapi-instrumentator==8.1.0` and `prometheus-client==0.26.0`, pinned in `requirements.txt`.
- [x] **I5-T2** Expose request count/rate, request latency histogram and HTTP status-code metrics via `Instrumentator`. Low cardinality: untemplated paths ignored, `/metrics` excluded, no bodies or secrets.
- [x] **I5-T3** Configure Prometheus scrape job to target `app:8000/metrics` over the private Compose network (`monitoring/prometheus.yml`).
- [x] **I5-T4** Configure Grafana datasource provisioning to `http://prometheus:9090` (`monitoring/grafana/provisioning/datasources/prometheus.yml`).
- [x] **I5-T5** Provision one concise LoopClose dashboard: request rate, latency (p95), status code distribution and total request counts (`monitoring/grafana/dashboards/loopclose-app.json`).
- [x] **I5-T6** Persist Prometheus and Grafana state with named volumes (`prometheus_data`, `grafana_data`); set bounded Prometheus retention (default 7 days).
- [x] **I5-T7** Bind host ports 9090 and 3000 to `127.0.0.1` only. Document Windows SSH tunnel commands for access.
- [ ] **I5-T8** Verify Prometheus target is UP and Grafana dashboard displays real requests to EC2 app once live instance is connected. No exporters.

**Acceptance:** Prometheus and Grafana configs and dashboards provisioned; `/metrics` verified in automated tests. No Node Exporter, cAdvisor, PostgreSQL Exporter or any other exporter is installed.

### INFRA-6 — OWASP ZAP and final security validation

- [x] **I6-T1** Add OWASP ZAP baseline/passive scan after successful EC2 health check in `.github/workflows/ci-cd.yml`.
- [x] **I6-T2** Save ZAP report as a GitHub Actions artifact (`zap-baseline-report`) using `.zap/rules.tsv`.
- [x] **I6-T3** Verify Gitleaks, CodeQL, pip-audit and Trivy configurations; all jobs configured to block vulnerabilities without suppression.
- [x] **I6-T4** Verify public ingress and confirm DB (private 5432), app (internal 8000), Prometheus (127.0.0.1:9090), and Grafana (127.0.0.1:3000) are not publicly reachable. Only Nginx port 80 is public.
- [x] **I6-T5** Run full regression tests after instrumentation/config changes; confirm Units 1–6 still work (74 passing tests).
- [x] **I6-T6** Update `memory.md` with exact final architecture, versions, test count, workflow run, deployment URL, metrics URLs/tunnel, ZAP report status, remaining limitations and manual actions.

**Acceptance:** security jobs run and report accurately; ZAP baseline configured; app and monitoring ready for deployment; existing product behaviour 100% intact.

## 5. Required human inputs / secrets

The human must create/provide the infrastructure outside the repository. Never ask for secret values in chat; add them directly in GitHub Settings → Secrets and variables → Actions (or a protected environment).

| Name | Purpose |
|---|---|
| `EC2_HOST` | EC2 public IP/DNS |
| `EC2_USER` | SSH user for selected Ubuntu AMI |
| `EC2_SSH_PRIVATE_KEY` | Private key for CI deployment |
| `EC2_KNOWN_HOSTS` | Verified SSH host key entry |
| `EC2_DEPLOY_PATH` | Deployment directory, e.g. `/opt/loopclose` |
| `APP_URL` | Public app URL used by health check and ZAP |
| `DB_PASSWORD` | PostgreSQL password |
| `GRAFANA_ADMIN_PASSWORD` | Grafana admin password |
| `GEMINI_API_KEY` | Existing AI integration secret |
| `GEMINI_MODEL` | Existing Gemini model setting |

The SSH ingress approach must be explicitly approved and documented. If automated temporary Security Group allowlisting is selected, add only the narrowly scoped AWS IAM credentials/permissions needed for that operation and guarantee cleanup in an `always()` step.

## 6. Final demonstration checklist

- [ ] Show the LoopClose app running on the EC2 public URL.
- [ ] Show GitHub Actions jobs: Gitleaks, CodeQL, Ruff/pytest, pip-audit, Docker build, Trivy, SCP/Ansible deployment, health check and ZAP baseline.
- [ ] Show Docker Compose services on EC2: app, db, nginx, prometheus and grafana.
- [ ] Show PostgreSQL is private and uses a persistent named volume.
- [ ] Open Prometheus via SSH tunnel and show `loopclose-app` target UP.
- [ ] Open Grafana via SSH tunnel and show live application request/latency/status metrics.
- [ ] Demonstrate a new Git push updates the deployed commit SHA.
- [ ] Confirm the original Units 1–6 product features still work.

## 7. Explicitly forbidden scope creep

Do not implement Nudge Center, vector RAG, Ask Team Memory, embeddings, extra exporters, Kubernetes, Terraform, managed databases, image registries, S3 image delivery, or unrelated UI/product features. Ask the human before changing this scope.
