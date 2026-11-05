# AGENTS.md — LoopClose AI Agent Operating Rules

> **MANDATORY: read all six project Markdown files completely before making any code or configuration change:** `memory.md`, `AGENTS.md`, `TASKS.md`, `PRD.md`, `ARCHITECTURE.md`, and `design.md`. Do not rely on chat summaries or assumptions. Re-read them at the start of every new work session/pass. The actual repository is the source of truth for what exists; document any discrepancy in `memory.md`.

Project: **LoopClose** — AI-powered meeting/handover, action-item tracking, knowledge-base linking and catch-up brief application. Product Units 1–6 are already implemented. The next work is a DevSecOps/infrastructure migration from Render to one AWS EC2 host.

---

## 1. Mandatory start-of-pass ritual

Perform these steps in order before editing anything:

1. Read `memory.md` completely enough to understand the Current State Snapshot, Open Questions, Next Work, Known Issues and the last three Pass Log entries. Read the whole file if practical; never skip the latest handover entry.
2. Read `AGENTS.md` completely.
3. Read `TASKS.md` completely. Identify the exact infrastructure task(s) authorised for this pass. Do not resume original product Units 7, 8 or 9.
4. Read `PRD.md`, `ARCHITECTURE.md` and `design.md` completely. These define product behaviour, technical architecture and UI constraints.
5. Inspect the repository: `git status`, `git log --oneline -10`, file tree, Dockerfile, Compose files, `.github/workflows`, `requirements*.txt`, `.env.example`, app config, health route, and tests. Do not assume the docs perfectly match the code.
6. Write a short working plan (3–6 lines): intended outcome, exact files to change, checks to run, and anything explicitly out of scope.
7. If a requirement conflicts with existing working behaviour, preserve Units 1–6 and raise the conflict in `memory.md`; do not silently remove functionality.

**No code edits are allowed until all six Markdown files have been read.**

## 2. Current project scope — important reset

- Treat **Units 1–6 as completed baseline functionality**. Preserve the existing LoopClose product: people, meetings, handovers, action items, history, carry-forward, AI recap, KB document summaries/linking (Level A), and catch-up briefs.
- **Skip original Units 7, 8 and 9 entirely.** Do not implement Nudge Center, nudge drafting/risk scoring, seed-data expansion/feature polish as a separate Unit 8, Level B vector RAG, embeddings, chunking, or Ask Team Memory. Do not invent replacement product features.
- The work after Unit 6 is a new **Infrastructure / DevSecOps workstream**, defined in `TASKS.md` and `ARCHITECTURE.md` §14.
- The production target is **one AWS EC2 Ubuntu instance** hosting the app, PostgreSQL, Nginx, Prometheus and Grafana using Docker Compose. The app and database are separate containers on the same EC2 host.
- The CI pipeline builds the application image on a GitHub Actions runner, scans it, exports it as a TAR archive and transfers it directly to EC2 using SSH/SCP. **Do not use ECR, Docker Hub, another registry, S3 image transfer, or a managed database.**
- Prometheus scrapes application-level FastAPI metrics directly from the app `/metrics` endpoint. Grafana reads Prometheus. **Do not add Node Exporter, cAdvisor, PostgreSQL Exporter, or any other exporter.**
- Security pipeline: Gitleaks, CodeQL, Ruff/pytest, Trivy image scan, pip-audit, and OWASP ZAP baseline scan after deployment. Keep the scans useful and do not suppress real findings merely to make CI green.
- Ansible is real automation: use it for repeatable EC2 provisioning/configuration and deployment orchestration. Do not create a playbook that is never invoked.

## 3. Scope discipline and deletion policy

- Work only on the infrastructure tasks explicitly selected in `TASKS.md`; do not bundle unrelated refactors.
- Preserve all working Units 1–6 behaviour and tests. Do not rewrite app architecture, UI, AI prompts or business logic for style.
- Remove obsolete Render-specific deployment configuration only after checking all references and confirming the EC2 path replaces it. This includes `render.yaml`, Render deploy hooks, Render-only environment variables/build args and stale Render documentation. Do not delete the app's `/health` endpoint or generic `GIT_SHA` support.
- Remove unimplemented Unit 7/9 stubs, routes, imports, dependencies or empty folders only after searching the repository and proving they are unused. Do not remove shared code or data structures needed by Units 1–6. Never delete a populated database, user data, Git history or logs as a cleanup shortcut.
- Do not add cloud services beyond those explicitly specified. In particular, no ECR, S3 image handoff, RDS, ECS, EKS, Kubernetes, Terraform, or managed database.
- No exporters: app-level metrics only.
- Do not invent successful deployments, cloud resource IDs, URLs, secrets, test results or versions. Record `NOT VERIFIED` when a check was not run.

## 4. Technology and implementation rules

- Preserve the existing stack: Python 3.12, FastAPI, SQLAlchemy 2.x, Pydantic v2, Jinja2, HTMX, PostgreSQL in production, SQLite for local/tests, Gemini through the existing AI abstraction.
- Docker Compose manages these services on EC2: `app`, `db`, `nginx`, `prometheus`, `grafana`. Do not put app and DB in one container.
- Use a named persistent Docker volume for PostgreSQL. The database must not publish port 5432 to the public host interface.
- Nginx is the public entry point (port 80; HTTPS only when a domain/certificate is actually configured). App port 8000 is internal to the Compose network.
- Bind Prometheus 9090 and Grafana 3000 to host loopback by default. Reach them with an SSH tunnel; do not expose monitoring interfaces to the public internet by default.
- Add only one small FastAPI Prometheus instrumentation dependency if the existing app has no suitable metrics endpoint. Verify current official documentation, pin the exact installed version, and document why it is needed. Expose request count, latency and status-code metrics; do not log request bodies or sensitive labels.
- Prometheus scrapes `http://app:8000/metrics` over the private Compose network. Grafana's provisioned Prometheus datasource uses `http://prometheus:9090`. Persist Grafana/Prometheus state with named volumes and set a modest Prometheus retention suitable for the EC2 disk.
- Build image on GitHub Actions, tag with the full commit SHA, scan the exact image, `docker save` it, compress it if useful, SCP it to EC2, then invoke Ansible to load and deploy it. No image registry.
- SSH must use a private key stored as a GitHub Actions secret and verified host keys (`known_hosts`). Never use `StrictHostKeyChecking=no`. Do not expose SSH to `0.0.0.0/0` as an undocumented default. Explain the GitHub-hosted runner IP allowlisting trade-off and use a narrowly scoped, human-approved access strategy.
- Ansible playbooks must be idempotent where practical, use `become` only where needed, avoid shell commands when a native module exists, and mark secret-bearing tasks `no_log: true`.
- Store secrets only in local `.env` (gitignored) or GitHub Actions secrets/environment secrets. The server-side `.env` must have restrictive permissions (`0600`) and be excluded from the image and Git.
- Use least-privilege GitHub Actions permissions, pin actions to verified stable major versions or immutable SHAs where practical, and use concurrency to prevent overlapping deployments.
- Do not guess package/action versions. Verify current official docs/release tags before adding or changing dependencies/actions and record exact choices in `memory.md`.

## 5. Application code rules (retain existing rules)

### 5.1 Layering
- Routers → services → models/AI/KB. No business logic in routers; no FastAPI/Jinja imports in services; no SQL in templates.
- All LLM access through `app/ai/client.py::AIClient`; KB retrieval through `app/kb/retrieval.py::find_related`.
- All settings through `app/config.py`; do not read `os.environ` elsewhere.

### 5.2 Style and quality
- Type hints on all function signatures; small, descriptive functions; docstrings on public service functions/classes.
- Ruff is the source of truth for lint and format. Respect the existing `pyproject.toml` configuration.
- No dead code, unexplained TODOs, commented-out blocks, or `print()` in application code; use structured logging.
- Store UTC, use timezone-aware datetimes, and preserve existing database semantics.

### 5.3 Database and AI safety
- SQLAlchemy 2.x typed models and parameterised queries. Existing `create_all()` approach is a known limitation; do not introduce Alembic unless explicitly requested.
- Never delete `action_item_events`; it is append-only history.
- AI outputs are validated, retried and fall back as already specified. Tests use `FakeProvider`; no tests call a real LLM or external network.
- Never log prompts, notes, KB content, model output, emails, API keys, tokens, passwords or `.env` values.

### 5.4 UI
- Follow `design.md` for any product UI change. Keep templates dumb, Jinja autoescape enabled, and never use `|safe` on user/AI content.
- Monitoring dashboards are Grafana dashboards, not new LoopClose application pages.

## 6. Required security behaviour

- Gitleaks scans repository history for secrets.
- CodeQL performs Python static analysis. Confirm repository visibility/plan supports code scanning; do not silently remove CodeQL if the repo is private and unsupported—report the limitation and ask for the required repository setting/plan.
- `pip-audit` checks Python dependencies. Report and assess findings; do not blindly upgrade major versions.
- Trivy scans the locally built image for OS and Python package vulnerabilities before transfer. Fail on configured HIGH/CRITICAL findings unless a documented, reviewed exception is approved by the human.
- OWASP ZAP runs a baseline passive scan against the deployed app after a successful health check. It is DAST, not a replacement for CodeQL or Trivy. Do not run active/destructive scans against production.
- Do not store or publish scan reports containing secrets or sensitive values.

## 7. Start and end verification

Before pushing code:

```powershell
.venv\Scripts\ruff format --check .
.venv\Scripts\ruff check .
.venv\Scripts\pytest -q
```

Also validate Docker Compose locally where possible:

```powershell
docker compose config
docker compose build
docker compose up -d
docker compose ps
```

For infra changes, run Ansible syntax validation and a check/diff run where safe:

```bash
ansible-playbook --syntax-check -i ops/ansible/inventory.ini ops/ansible/playbooks/site.yml
```

Do not claim EC2 deployment, metrics scraping, dashboard provisioning, ZAP or workflow success unless actually verified. A failed or unverified external prerequisite must be clearly stated.

## 8. Mandatory end-of-pass ritual and memory update

1. Run relevant Ruff and pytest checks; run Docker/Ansible checks for changed infrastructure.
2. Verify the exact acceptance criteria in `TASKS.md`; tick only tasks actually completed and verified.
3. Append a new entry to `memory.md` using its exact pass-entry template. **Never delete or rewrite old Pass Log entries.**
4. Rewrite the living sections of `memory.md`: Current State Snapshot, File Registry, Dependencies, Environment & Deployment, Decisions, Open Questions, Next Work, Known Issues.
5. Record exact commands, outputs/results, files changed, dependencies/action versions, secrets the human must configure (names only), and anything not verified.
6. Keep the repository runnable. Do not commit or push unless the human has authorised that action; report the current Git state.
7. Final response must distinguish completed, tested, deployed, and pending manual steps.

## 9. Secrets and human-only actions

The agent cannot create the user's AWS account/EC2 instance, GitHub repository, SSH keypair, domain, or GitHub secrets without explicit supported access and authorisation. List exact human actions and secret names; never ask the user to paste private keys or passwords into chat.

Expected GitHub Actions secret names will be finalised in `ARCHITECTURE.md` §14 and `memory.md`. Never write their values into Markdown.

## 10. Quick commands

```powershell
# Windows local app
.venv\Scripts\activate
pip install -r requirements.txt -r requirements-dev.txt
uvicorn app.main:app --reload
.venv\Scripts\ruff format --check .
.venv\Scripts\ruff check .
.venv\Scripts\pytest -q

# Local containers
docker compose config
docker compose up -d --build
docker compose ps
docker compose logs -f app

# SSH tunnel to EC2-hosted monitoring (run in PowerShell)
ssh -i path\to\ec2-key.pem -L 3000:127.0.0.1:3000 -L 9090:127.0.0.1:9090 ubuntu@EC2_PUBLIC_IP
# Then browse http://localhost:3000 (Grafana) and http://localhost:9090 (Prometheus)
```
