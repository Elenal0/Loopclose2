# ARCHITECTURE.md — LoopClose

> **Audience:** any AI coding agent (or human) picking up this project mid-way.
> **Authority:** This file is the source of truth for *technology, structure, data model, APIs, AI design, logging and DevOps*. For product behaviour see `PRD.md`; for visuals see `design.md`; for rules see `AGENTS.md`; for progress see `TASKS.md` and `memory.md`.
> **Project codename:** LoopClose ("close the loop on every open item"). The name can be changed later; do not rename it in code without a human instruction.

---

## 1. Purpose and scope

LoopClose is an AI-assisted meeting and work-handover application. The product baseline is **complete through original Units 1–6**:

1. People, meetings and handovers (agenda, participants, attendance, notes and decisions).
2. Action items with owner, due date, status, outcome and append-only history.
3. Carry-forward of open work between meetings and bulk handover reassignment.
4. AI recap proposals with human review/apply and deterministic fallback.
5. Knowledge base documents, AI summaries and **Level A** LLM-based related-knowledge linking with keyword fallback.
6. Personalised catch-up briefs with template fallback.

**Explicitly out of scope:** original Unit 7 Nudge Center; original Unit 8 seed-data/README/polish pass as a separate feature unit; original Unit 9 Level B vector RAG, embeddings, chunking and Ask Team Memory. Do not implement these unless the human changes scope.

The remaining work is infrastructure and DevSecOps: migrate deployment from EC2 to a single AWS EC2 Ubuntu instance; run the app and PostgreSQL in separate Docker containers on that same instance; use GitHub Actions to test, scan, build and transfer the image directly to EC2 via SSH/SCP; use Ansible for provisioning/deployment; and run Prometheus + Grafana for **application-level metrics only**.

## 2. System overview

### 2.1 Target deployment architecture

```text
Developer (Windows)
  | git push
  v
GitHub repository
  |
  v
GitHub Actions CI/CD
  |-- Gitleaks (secret scanning)
  |-- CodeQL (Python SAST)
  |-- Ruff + pytest
  |-- pip-audit (Python dependency audit)
  |-- Docker build on GitHub runner
  |-- Trivy scan of the exact built image
  |-- docker save -> compressed image TAR
  |-- SCP over verified SSH to EC2 (no registry / no S3 image transfer)
  |-- Ansible playbook: docker load + docker compose up -d
  |-- health check (/health, expected commit SHA)
  `-- OWASP ZAP baseline scan after successful deployment

AWS EC2 (Ubuntu; one host)
  Docker Compose private network
    |-- nginx       public entry point: host port 80 (443 only when TLS is configured)
    |-- app         FastAPI + Jinja2/HTMX; internal port 8000; /metrics
    |-- db          PostgreSQL; internal port 5432; persistent named volume
    |-- prometheus  scrapes app:8000/metrics; host port 9090 bound to loopback
    `-- grafana     Prometheus datasource; host port 3000 bound to loopback

Browser -> EC2 public IP/domain -> Nginx -> FastAPI app -> PostgreSQL
Prometheus -> FastAPI /metrics -> Grafana dashboards
Gemini API is an outbound HTTPS dependency only when configured.
```

### 2.2 Network and persistence boundaries

- All Compose services share a private Docker network. PostgreSQL is reachable by the app as `db:5432`; never publish port 5432 to the public host interface.
- Nginx is the only public application entry point. It proxies to `app:8000`. Do not publish the app container port directly to the public interface.
- Prometheus scrapes `http://app:8000/metrics` over the Compose network. Grafana queries `http://prometheus:9090` over the same network.
- Bind Prometheus `9090` and Grafana `3000` to `127.0.0.1` on the EC2 host by default. The operator views them through an SSH tunnel; do not expose these admin interfaces publicly by default.
- PostgreSQL uses a named persistent Docker volume. App container replacement must not delete database data. Backups are a separate operational safeguard; a Docker volume is not a backup.
- This is a single-host design: EC2 failure or disk loss affects app, DB and monitoring together. It is suitable for a learning/demo deployment, not high availability.

### 2.3 Key decisions

| Decision | Reason |
|---|---|
| Preserve the single FastAPI deployable | Existing Units 1–6 serve both HTML and JSON; avoid unnecessary rewrite. |
| App and PostgreSQL are separate containers on one EC2 | Independent lifecycle and persistent DB storage without a managed database. |
| Docker Compose | Simple multi-container orchestration on one host; Kubernetes is unnecessary. |
| Direct image TAR transfer by SCP | Explicit requirement: no ECR, Docker Hub, registry or S3 image handoff. |
| Ansible provisions and deploys | Repeatable server setup and deployment; not merely an unused playbook. |
| Nginx in front of app | Single public ingress and future TLS termination. |
| Prometheus + Grafana only | Small application-level observability stack; no exporters. |
| PostgreSQL in production, SQLite locally/tests | Preserve existing SQLAlchemy portability. |
| Keep Level A KB linking | It is part of completed Unit 6; do not add vector RAG. |

## 3. Technology stack and specifications

Pin exact dependency versions after installing and verifying them. Never invent versions. Verify GitHub Action tags and third-party APIs against current official documentation before use.

| Layer | Technology | Specification |
|---|---|---|
| Language/runtime | Python 3.12 | Preserve existing application runtime. |
| Web | FastAPI + Uvicorn | One worker initially; internal container port 8000. |
| UI | Jinja2 + HTMX + existing CSS | Preserve current server-rendered UI. |
| ORM/database | SQLAlchemy 2.x + PostgreSQL | `psycopg` driver; PostgreSQL container on the same EC2 host. |
| Local/tests DB | SQLite | Existing test isolation and local development behaviour. |
| AI | Existing Gemini provider abstraction | Keep existing recap, KB summary/link and brief features. No new AI operations. |
| Retrieval | Level A LLM catalogue linking + keyword fallback | No embeddings, vector store, chunking or Ask Team Memory. |
| Metrics instrumentation | FastAPI Prometheus instrumentation library | Add one small dependency only if no existing `/metrics` endpoint exists; pin exact version. No exporters. |
| Containers | Docker Engine + Docker Compose plugin | App, DB, Nginx, Prometheus and Grafana services. |
| Reverse proxy | Nginx | Public port 80; optional 443 only with configured TLS/domain. |
| Cloud host | AWS EC2 Ubuntu | One instance hosts all containers and persistent DB volume. |
| CI/CD | GitHub Actions | Test, scan, build, save TAR, SCP and run Ansible. |
| Secrets scan | Gitleaks | Full-history scan. |
| SAST | CodeQL | Python analysis; verify repository eligibility. |
| Dependency audit | pip-audit | Audit Python dependencies. |
| Image scan | Trivy | Scan the exact locally built image before transfer. |
| DAST | OWASP ZAP baseline | Passive baseline scan after deployment/health check. |
| Configuration management | Ansible | Provision Docker/Compose/Nginx prerequisites and deploy Compose stack. |
| Metrics | Prometheus | Scrape only the FastAPI `/metrics` endpoint. |
| Dashboards | Grafana | Prometheus datasource and a small application metrics dashboard. |

**Explicitly excluded:** ECR, Docker Hub and all image registries; S3 image transfer; RDS or any managed DB; ECS/EKS/Kubernetes; Terraform; Node Exporter; cAdvisor; PostgreSQL Exporter; other exporters; Nudge Center; vector RAG/Ask Team Memory.

## 4. Repository structure

```
loopclose/
├── AGENTS.md            # rules for AI agents
├── TASKS.md             # unit-by-unit checklist
├── PRD.md               # product requirements
├── ARCHITECTURE.md      # this file
├── design.md            # UI/UX spec
├── memory.md            # agent handover log (append-only)
├── README.md            # existing project instructions; update only where deployment commands change
├── Dockerfile
├── .dockerignore
├── .gitignore
├── .gitleaks.toml       # gitleaks allowlist (placeholders only)
├── .env.example         # variable NAMES + dummy values only
├── requirements.txt
├── requirements-dev.txt
├── pyproject.toml       # ruff + pytest config
├── .github/
│   └── workflows/
│       └── ci-cd.yml    # the pipeline
├── app/
│   ├── __init__.py
│   ├── main.py              # app factory, router registration, startup (create_all, optional seed)
│   ├── config.py            # Settings (pydantic-settings), DB URL normalisation
│   ├── database.py          # engine, SessionLocal, Base, get_db dependency
│   ├── logging_config.py    # JSON formatter, handlers, redaction filter, request_id contextvar
│   ├── middleware.py        # RequestContextMiddleware (request ID, access log)
│   ├── errors.py            # custom exceptions + global handlers (HTML + JSON)
│   ├── enums.py             # all enums (single place)
│   ├── models/              # SQLAlchemy models (one file per aggregate)
│   │   ├── person.py  meeting.py  action_item.py  decision.py
│   │   ├── kb.py (documents and links)  brief.py
│   ├── schemas/             # Pydantic request/response + AI output schemas
│   ├── services/            # business logic (NO FastAPI imports here)
│   │   ├── people_service.py
│   │   ├── meeting_service.py        # CRUD, lifecycle, participants, handover reassign
│   │   ├── action_item_service.py    # CRUD, transitions, reassign, close/outcome, events
│   │   ├── carry_forward_service.py
│   │   ├── recap_service.py          # AI recap + apply
│   │   ├── brief_service.py          # catch-up brief
│   │   ├── kb_service.py             # documents, summaries, linking orchestration
│   ├── ai/
│   │   ├── provider.py       # LLMProvider protocol (generate_json)
│   │   ├── gemini.py         # GeminiProvider
│   │   ├── fake.py           # FakeProvider
│   │   ├── client.py         # AIClient: timeout, retry, JSON validation, logging, error mapping
│   │   ├── fallbacks.py      # heuristic recap, template brief, keyword linker
│   │   └── prompts/          # recap, brief, kb_summary and link
│   ├── kb/
│   │   └── retrieval.py      # Level A LLM + keyword fallback + find_related()
│   ├── routers/
│   │   ├── web_dashboard.py  web_meetings.py  web_items.py  web_kb.py  web_people.py
│   │   ├── api_v1.py         # thin JSON wrappers over services
│   │   └── health.py
│   ├── templates/
│   │   ├── base.html  dashboard.html  error.html
│   │   ├── meetings/ (list, new, detail, _tabs ...)
│   │   ├── items/  kb/  people/
│   │   └── partials/ (badges, toasts, item_row, ai_banner, empty_state ...)
│   └── static/
│       ├── css/app.css
│       └── js/app.js
├── scripts/
│   └── seed.py              # demo data (idempotent)
├── tests/
│   ├── conftest.py          # temp DB, TestClient, FakeProvider fixture
│   └── test_*.py
├── compose.yaml             # app + db + nginx + prometheus + grafana
├── nginx/                    # reverse proxy configuration
├── monitoring/               # prometheus.yml and Grafana provisioning/dashboard files
├── ops/ansible/              # inventory, playbooks, roles/templates for EC2
└── logs/                    # runtime log files (gitignored)
```

---

## 5. Configuration (environment variables)

All application config is read through `app/config.py`. `.env.example` contains names and dummy placeholders only. The production `.env` is created on EC2 by the deployment/provisioning process with restrictive permissions; it is never copied into the Docker image or committed.

| Variable | Default / source | Purpose |
|---|---|---|
| `APP_ENV` | `local`; EC2: `production` | Runtime mode. |
| `APP_NAME` | `LoopClose` | Display name. |
| `DATABASE_URL` | Local SQLite; EC2 Compose value | `postgresql+psycopg://<user>:<password>@db:5432/<database>` on EC2. URL-encode reserved password characters. |
| `LLM_PROVIDER` | `gemini` | Existing `gemini` or `fake` provider. |
| `GEMINI_API_KEY` | Secret | Gemini API key; never log. |
| `GEMINI_MODEL` | Explicit environment value | Existing text/JSON model. |
| `AI_TIMEOUT_SECONDS` | `25` | Per-call timeout. |
| `AI_MAX_RETRIES` | `1` | Existing retry policy. |
| `AI_MAX_INPUT_CHARS` | `12000` | Existing prompt truncation limit. |
| `RETRIEVER` | `llm` | Level A `llm` with keyword fallback only. |
| `LOG_LEVEL` | `INFO` | Structured application log level. |
| `LOG_FILE` | `logs/app.log` | Rotating log file; stdout is also emitted for `docker compose logs`. |
| `SEED_DEMO_DATA` | `false` | Preserve existing setting; do not expand seed scope as original deferred legacy polish scope. |
| `GIT_SHA` | Build-time commit SHA | Exposed by `/health` for deployment verification. |
| `DB_NAME` / `DB_USER` / `DB_PASSWORD` | Server `.env` | PostgreSQL container initialisation; secret password. |
| `GRAFANA_ADMIN_USER` / `GRAFANA_ADMIN_PASSWORD` | Server `.env` | Grafana initial admin credentials. |
| `PROMETHEUS_RETENTION` | `7d` | Keep monitoring storage bounded for a small EC2 disk. |

GitHub Actions secrets/environment values (names only): `EC2_HOST`, `EC2_USER`, `EC2_SSH_PRIVATE_KEY`, `EC2_KNOWN_HOSTS`, `EC2_DEPLOY_PATH`, `APP_URL`, `GEMINI_API_KEY`, `GEMINI_MODEL`, `DB_PASSWORD`, `GRAFANA_ADMIN_PASSWORD`. Add AWS credentials only if the human explicitly chooses automated temporary Security Group allowlisting; do not add them by default. Never store values in Markdown.

## 6. Layering rules (must be respected)

```
routers (web/api)  →  services  →  models / ai / kb
      ↑ only HTTP concerns      ↑ business rules, transactions, event logging
```

1. **Routers** parse/validate input (Pydantic), call a service, render a template or return JSON. No business rules, no direct SQL.
2. **Services** own business rules, state-machine checks, transactions, history events and domain log events. They never import FastAPI or Jinja.
3. **AI code** is reachable only through `app/ai/client.py` (`AIClient`). Services call `AIClient`, never a vendor SDK directly.
4. **Retrieval** is reachable only through `app/kb/retrieval.py::find_related`.
5. **Models** contain no logic beyond simple properties (e.g. `is_overdue`).
6. Every service method that mutates state writes an `action_item_events` row (for items) and emits a structured log event.

---

## 7. Data model

All tables have `id` (integer PK, autoincrement) and `created_at` (UTC, default now) unless stated. `updated_at` on mutable aggregates. Use `Mapped[]` typed columns. Enums are stored as strings (`native_enum=False`) so SQLite and Postgres behave the same.

### 7.1 Tables

**people**
| column | type | notes |
|---|---|---|
| name | str(100) | required |
| email | str(200) | optional, unique if present |
| role_title | str(100) | optional |
| is_active | bool | default true (false = "on leave"/left; still shown on history) |

**meetings** (a *meeting* or a *work handover*; one table, `kind` distinguishes)
| column | type | notes |
|---|---|---|
| kind | enum `MeetingKind` | `meeting` \| `handover` |
| title | str(200) | required, ≥3 chars |
| status | enum `MeetingStatus` | default `planned` |
| scheduled_at | datetime | required |
| agenda | text | optional, ≤5,000 chars |
| raw_notes | text | optional, ≤50,000 chars |
| summary | text | optional (AI or manual) |
| ai_status | enum `AiStatus` | `none` \| `ok` \| `fallback` \| `failed` |
| ai_error | str(300) | short human-readable reason, never a stack trace |
| recap_json | JSON | last AI recap *proposal* (before apply) |
| previous_meeting_id | FK meetings.id | nullable; series link / carry-forward source |
| handover_from_id | FK people.id | nullable; required when `kind=handover` |
| handover_to_id | FK people.id | nullable; required when `kind=handover`, must differ from `from` |
| closed_at | datetime | nullable |

**meeting_participants** — `meeting_id` FK, `person_id` FK, `attendance` enum `Attendance` (`invited`, `attended`, `absent`, `no_response`, `declined`). Unique `(meeting_id, person_id)`.

**decisions** — `meeting_id` FK (cascade), `text` str(1000), `source` enum `ItemSource` (`manual`|`ai`).

**action_items**
| column | type | notes |
|---|---|---|
| title | str(200) | required, ≥3 chars |
| description | text | optional |
| origin_meeting_id | FK meetings.id | required; where it was first raised (never changes) |
| current_meeting_id | FK meetings.id | nullable; whose agenda it is on now (changes on carry-forward) |
| owner_id | FK people.id | nullable (unassigned allowed but flagged) |
| due_date | date | nullable |
| priority | enum `Priority` | `low`\|`medium`\|`high`, default `medium` |
| status | enum `ActionStatus` | `open`\|`in_progress`\|`blocked`\|`done`\|`dropped` |
| blocked_reason | str(300) | required when status=`blocked` |
| outcome_type | enum `OutcomeType` | nullable until closed |
| outcome_note | text | required (≥5 chars) when closed |
| source | enum `ItemSource` | `manual`\|`ai`\|`heuristic` |
| carried_count | int | default 0, incremented on each carry-forward |
| closed_at | datetime | nullable |

**action_item_events** (append-only history; never updated or deleted)
| column | type | notes |
|---|---|---|
| action_item_id | FK | indexed |
| event_type | enum `EventType` | see 8.2 |
| actor | str(50) | `user` \| `ai` \| `system` (no auth in this app) |
| from_value / to_value | str(200) | e.g. owner names, statuses, meeting ids |
| meeting_id | FK | nullable context |
| note | text | optional |
| request_id | str(32) | correlation ID of the request that caused it |

**kb_documents** — `title` str(200), `source_type` enum `KBSource` (`note`, `document`, `chat`, `email`, `transcript`, `meeting`), `content` text (≤100,000 chars), `summary` text (AI or fallback), `tags` JSON list[str], `meeting_id` FK nullable (set for auto-indexed meeting docs; at most one doc per meeting), `index_status` enum `IndexStatus` (`pending`, `indexed`, `failed`, `not_applicable`), `updated_at`.

**kb_links** — `entity_type` enum `EntityType` (`action_item`|`decision`|`meeting`), `entity_id` int, `document_id` FK, `score` float (0–1), `reason` str(300), `method` enum `LinkMethod` (`llm`|`keyword`|`manual`). Unique `(entity_type, entity_id, document_id)`.

**catchup_briefs** — `meeting_id` FK, `person_id` FK, `content_json` JSON (schema in 10.4), `generated_by` enum `GeneratedBy` (`ai`|`fallback`).


### 7.2 Indexes
`action_items(status, due_date)`, `action_items(owner_id)`, `action_items(current_meeting_id)`, `action_item_events(action_item_id, created_at)`, `meeting_participants(person_id)`, `kb_links(entity_type, entity_id)`.

### 7.3 Computed properties
- `ActionItem.is_overdue` = `due_date < today AND status in (open, in_progress, blocked)`
- `ActionItem.is_open` = status in (`open`, `in_progress`, `blocked`)

---

## 8. State machines and history events

### 8.1 Meeting status
```
planned ──► held ──► closed
   │          │
   └──► cancelled ◄┘      (cancelled and closed are terminal)
```
- `held → closed` allowed any time; open items are **not** auto-closed. The UI then prompts for carry-forward.
- Notes/agenda editable while `planned` or `held`; read-only when `closed`/`cancelled`.
- AI recap allowed when status in (`planned`, `held`) and `len(raw_notes) ≥ 20`.
- Invalid transition → `InvalidTransitionError` (HTTP 409 / inline error).

### 8.2 Action item status
```
open ⇄ in_progress ⇄ blocked          (blocked requires blocked_reason)
  └────────┬────────────┘
           ▼
  close(outcome_type, outcome_note)  ──►  done | dropped
  reopen: done | dropped ──► open
```
- `POST /items/{id}/close` is the **only** way to reach `done`/`dropped`; it requires `outcome_type` and `outcome_note` (≥5 chars). This satisfies "record the outcome of each item".
- `outcome_type → status` mapping (final): `completed` → `done`, `partially_completed` → `done`, `dropped` → `dropped`, `superseded` → `dropped`.
- On **reopen**, the item's `status` returns to `open` and `closed_at`, `outcome_type`, `outcome_note` are cleared on the row; the previous outcome is preserved in the `reopened` history event (`note` field) so history is never lost.

### 8.3 `EventType` values
`created`, `updated`, `reassigned`, `status_changed`, `due_date_changed`, `carried_forward`, `blocked`, `closed`, `reopened`, `outcome_recorded`, `kb_linked`.

Every one of these also emits a log event named `item.<event_type>` (see Section 12).

---

## 9. Routes

HTML routes return a **full page** normally, and a **partial** when the request has header `HX-Request: true` (HTMX). Routes live in `app/routers/web_*.py`. JSON equivalents (thin wrappers) live in `api_v1.py` under `/api/v1` and are *lower priority than the HTML UI*; implement them alongside each unit if time allows and log skipped ones in `memory.md`.

### 9.1 Web routes
| Method | Path | Purpose |
|---|---|---|
| GET | `/` | Dashboard: my open items, overdue, upcoming meetings, recent decisions and needs-owner count |
| GET | `/people` · POST `/people` | List / create people |
| GET | `/meetings` | List with filters (kind, status) |
| GET | `/meetings/new` · POST `/meetings` | Create meeting / handover (+ participants, + optional carry-forward from previous) |
| GET | `/meetings/{id}` | Detail with tabs: Overview · Notes & Recap · Items · Brief · Knowledge · History |
| POST | `/meetings/{id}` | Update agenda/title/schedule |
| POST | `/meetings/{id}/status` | Lifecycle transition |
| POST | `/meetings/{id}/participants` | Add/update participant & attendance |
| POST | `/meetings/{id}/notes` | Save raw notes |
| POST | `/meetings/{id}/recap` | **AI**: generate recap proposal (HTMX partial) |
| POST | `/meetings/{id}/recap/apply` | Apply selected proposal rows → creates decisions + items |
| POST | `/meetings/{id}/decisions` · POST `/decisions/{id}/delete` | Manual decision CRUD |
| POST | `/meetings/{id}/carry-forward` | Carry open items to `to_meeting_id` |
| POST | `/meetings/{id}/handover/reassign` | Bulk reassign `from` person's open items to `to` person |
| GET/POST | `/meetings/{id}/brief` | View / **AI**: generate catch-up brief for `person_id` |
| POST | `/meetings/{id}/items` | Create action item |
| GET | `/items` | Filterable list (status, owner, overdue, meeting) |
| GET | `/items/{id}` | Detail + history timeline + linked knowledge |
| POST | `/items/{id}` | Edit title/description/due/priority |
| POST | `/items/{id}/status` | open / in_progress / blocked / reopen |
| POST | `/items/{id}/reassign` | Change owner (event `reassigned`) |
| POST | `/items/{id}/close` | Close with outcome |
| POST | `/items/{id}/suggest-links` | **AI/RAG**: find related KB (HTMX partial) |
| POST | `/links` · POST `/links/{id}/delete` | Manual KB link add/remove |
| GET | `/kb` · GET `/kb/new` · POST `/kb` | List / add document (AI summary + tags on create) |
| GET | `/kb/{id}` | Document + "referenced by" |
| POST | `/kb/{id}/reindex` | Re-summarise document metadata (no vector embedding) |
| GET | `/health` | JSON health (no auth) | | JSON health (no auth) |

### 9.2 JSON API (`/api/v1`)
`GET/POST /meetings`, `GET /meetings/{id}`, `POST /meetings/{id}/recap`, `GET/POST /action-items`, `GET /action-items/{id}`, `POST /action-items/{id}/close`, `POST /action-items/{id}/reassign`, `GET/POST /kb/documents`, `GET /kb/search?q=`, `POST /meetings/{id}/brief`. Error shape:
```json
{"error": {"code": "validation_error", "message": "Title must be at least 3 characters", "request_id": "a1b2c3...", "details": [{"field": "title", "issue": "too_short"}]}}
```

### 9.3 `/health` response
```json
{"status": "ok", "app": "LoopClose", "env": "production", "commit": "<sha or null>", "db": "ok", "ai": {"provider": "gemini", "configured": true}}
```
`db` is verified with `SELECT 1`. `status` = `degraded` if DB check fails (HTTP 503). AI being unconfigured does **not** fail health (the app degrades gracefully).

---

## 10. AI subsystem

### 10.1 Components
- `LLMProvider` (protocol): `generate_json(system, prompt, schema_hint, timeout) -> str`. No embedding method is required.
- `GeminiProvider`, `FakeProvider` implement it. Verify the `google-genai` SDK usage against **current official docs** before coding (APIs change).
- `AIClient` (the only thing services use) provides:
  `run_structured(operation: str, system: str, prompt: str, model_cls: type[BaseModel]) -> AIResult[T]`
  which: truncates input → calls provider with timeout → strips code fences → `json.loads` → validates with Pydantic → on failure retries up to `AI_MAX_RETRIES` (with a "return valid JSON only" reminder on JSON errors) → returns `AIResult(ok, data, error_code, latency_ms, attempts)`. **It never raises to the caller for AI problems**; callers inspect `ok` and use a fallback.
- `fallbacks.py`: deterministic degraded behaviour for each operation (see 10.3).

### 10.2 Operations
| Operation key | Input | Output model | Used by |
|---|---|---|---|
| `recap` | agenda + raw notes + participant names + meeting date | `RecapResult` | recap_service |
| `brief` | structured context (meeting summary, decisions, person's items, history deltas, related KB) | `BriefResult` | brief_service |
| `kb_summarize` | document text | `{summary, tags[]}` | kb_service |
| `link` | entity text + KB catalogue (id, title, summary) | `{matches:[{document_id, relevance, reason}]}` | retrieval (Level A) |

### 10.3 Failure handling per operation (mandatory)
| Operation | Fallback when AI fails or key missing | UI behaviour |
|---|---|---|
| `recap` | **Heuristic recap**: lines starting with `Action:`, `TODO`, `- [ ]`, `AI:` become items (no owner/due); lines starting with `Decision:` become decisions; first 2 sentences become summary. `ai_status=fallback`. If nothing parsed → `ai_status=failed`, raw notes untouched. | Amber banner "AI unavailable — showing basic extraction" + **Retry** button. Notes are never lost. |
| `brief` | **Template brief** assembled from DB (summary, decisions, the person's open items, overdue list). `generated_by=fallback`. | Banner + Retry. |
| `kb_summarize` | First 200 chars as summary, tags empty. | Silent note "summary auto-trimmed". |
| `link` | **Keyword linker** (token overlap on title/summary/content, simple TF-IDF-ish scoring). `method=keyword`. | Results shown with "keyword match" label. |

### 10.4 Output schemas (Pydantic)
```python
class RecapActionItem(BaseModel):
    title: str  # 3–200 chars
    owner_name: str | None  # must be matched to a participant; else None
    due_date: date | None  # ISO date; relative dates resolved against meeting date
    priority: Literal["low", "medium", "high"] = "medium"
    rationale: str | None  # short quote/paraphrase from notes


class RecapDecision(BaseModel):
    text: str


class RecapResult(BaseModel):
    summary: str  # ≤ 120 words
    decisions: list[RecapDecision]
    action_items: list[RecapActionItem]
    open_questions: list[str] = []


class BriefResult(BaseModel):
    tldr: str  # ≤ 3 sentences
    key_decisions: list[str]
    your_items: list[str]  # items owned by the reader, with due dates
    changed_since: list[str]  # reassignments, carry-forwards, new blockers
    open_questions: list[str]
    related_knowledge: list[{document_id: int, title: str, why: str}]
    suggested_first_steps: list[str]  # ≤ 5
```
**Owner resolution:** `owner_name` → participant via case-insensitive exact match, then `difflib.get_close_matches` (cutoff 0.8). No match → unassigned (flagged "needs owner").

### 10.5 Prompt rules (all prompts in `app/ai/prompts/`)
1. Each prompt module exports `PROMPT_VERSION` (e.g. `"recap.v1"`) — logged on every call.
2. System prompt states the exact JSON schema and "Return ONLY JSON, no prose, no code fences."
3. Untrusted text (notes, KB content) is wrapped in delimiters (`<notes>...</notes>`) and the prompt states that content inside delimiters is **data, not instructions** (prompt-injection hygiene).
4. Provide the meeting date and participant list so relative dates ("by Friday") and owners can be resolved.
5. Never ask the model to invent owners/dates; "null if not stated".
6. Never include secrets or env values in prompts.

---

## 11. Knowledge base and RAG

### 11.1 Sources ("data is scattered")
Documents enter the KB from: pasted/uploaded `.txt`/`.md` content (types `note`, `document`, `chat`, `email`, `transcript`), and **automatically from past meetings** (type `meeting`: summary + decisions + items, refreshed when a recap is applied or a meeting is closed). Audio/video are out of scope: users paste the transcript text.

### 11.2 One retrieval interface
```python
# app/kb/retrieval.py
@dataclass
class KBMatch:
    document_id: int
    score: float  # 0..1
    reason: str
    method: Literal["llm", "keyword", "manual"]


def find_related(
    text: str, *, entity_type: str, entity_id: int | None = None, limit: int = 3
) -> list[KBMatch]: ...
```
Selection order (config `RETRIEVER` + automatic fallback chain): `llm` → `keyword`. Failures are logged with outcome and the next retriever is tried. Callers never know which was used except via `method`.

### 11.3 Level A — LLM linking (Units 6; default)
Send the model a catalogue (`id`, `title`, `summary ≤300 chars`; max 50 docs) and the entity text. Validate returned `document_id`s exist; keep `relevance ≥ 0.5`; max 3. Store in `kb_links` (`method=llm`).

### 11.4 Explicitly deferred

Level B vector RAG, embeddings, chunking, vector storage and Ask Team Memory are not part of this project scope. Do not add `kb_chunks`, embedding code or an embedding dependency.

### 11.5 Rules
- Every link shows its **source document title** and **method**.
- Links are suggestions; the user can remove them (`manual` links are never auto-removed).
- Re-linking an entity replaces its non-manual links only.

---

## 12. Logging and observability

### 12.1 Setup (`app/logging_config.py`)
- One JSON object per line (JSON Lines). Handlers: **rotating file** (`LOG_FILE`, 5 MB × 3) **and stdout** (Docker captures stdout; use `docker compose logs` for runtime inspection).
- Levels: `DEBUG` (dev detail), `INFO` (normal events), `WARNING` (degraded/fallback/validation rejects), `ERROR` (unhandled errors, AI hard failures), `CRITICAL` (startup cannot continue).
- **Correlation ID:** `RequestContextMiddleware` reads inbound `X-Request-ID` or generates `uuid4().hex[:16]`, stores it in a `contextvars.ContextVar`, adds it to every log record via a logging filter, and returns it in the `X-Request-ID` response header. Error pages display it.
- **Redaction filter:** masks values for keys matching `key|token|secret|password|authorization|api_key` and any string equal to a configured secret value. Prompts/notes content is **not** logged (log lengths and counts instead). No emails in logs (log person IDs).

### 12.2 Standard fields
`ts` (ISO-8601 UTC), `level`, `logger`, `event`, `request_id`, plus event-specific fields. Example lines:
```json
{"ts":"2026-10-30T09:15:02.113Z","level":"INFO","logger":"app.http","event":"http.request","request_id":"9f2c1a7d3b4e5f60","method":"POST","path":"/meetings/4/recap","status":200,"duration_ms":2341}
{"ts":"2026-10-30T09:15:02.101Z","level":"INFO","logger":"app.ai","event":"ai.call","request_id":"9f2c1a7d3b4e5f60","operation":"recap","provider":"gemini","model":"<model>","prompt_version":"recap.v1","latency_ms":2210,"attempts":1,"outcome":"success","input_chars":3120,"output_chars":1480}
{"ts":"2026-10-30T09:16:10.882Z","level":"INFO","logger":"app.items","event":"item.closed","request_id":"c01d...","item_id":12,"outcome_type":"completed","meeting_id":4}
{"ts":"2026-10-30T09:17:44.310Z","level":"WARNING","logger":"app.ai","event":"ai.fallback","request_id":"77ab...","operation":"brief","reason":"timeout","fallback":"template"}
```

### 12.3 Event catalogue (minimum)
| Group | Events |
|---|---|
| HTTP | `http.request` (method, path, status, duration_ms), `http.error` |
| Meetings | `meeting.created`, `meeting.updated`, `meeting.status_changed`, `meeting.participant_added`, `meeting.notes_saved`, `meeting.carry_forward` (count), `meeting.handover_reassign` (count) |
| Items | `item.created`, `item.updated`, `item.reassigned` (from_owner_id,to_owner_id), `item.status_changed`, `item.blocked`, `item.closed` (outcome_type), `item.reopened`, `item.carried_forward` |
| AI | `ai.call` (operation, provider, model, prompt_version, latency_ms, attempts, outcome=`success`\|`retry_success`\|`fallback`\|`error`, error_code, input_chars, output_chars), `ai.fallback`, `ai.truncated_input` |
| KB | `kb.document_created`, `kb.summarized`, `kb.indexed` (chunks), `kb.link_created`, `kb.retrieval` (retriever, matches, latency_ms) |
| System | `app.startup` (env, db_dialect, ai_provider configured yes/no — never the key), `app.seeded`, `app.error` (exception type, message, stack in `exc_info` at ERROR) |

### 12.4 Errors with context
Unhandled exceptions are logged at ERROR with `exc_info`, `request_id`, route, and relevant IDs, then rendered as a friendly error page/JSON with the request ID.

---

## 13. Validation, errors, security

**Validation (Pydantic + service rules)** — see `PRD.md` §8 for the full rule list. Key ones: title 3–200 chars; due date cannot be set in the past on creation; owner must exist and be active; handover requires distinct from/to people; outcome required to close; blocked requires reason; notes length caps; enum values validated; HTML form errors are returned inline per field (HTTP 422 for API, re-rendered form with messages for HTML).

**Exceptions** (`app/errors.py`): `NotFoundError` (404), `ValidationFailed` (422), `InvalidTransitionError` (409), `AIUnavailableError` (internal only; never surfaces as 500 — becomes fallback). Global handlers return HTML (error page, or HTMX toast partial) or JSON based on `Accept`/path.

**Security basics**
- No secrets in code, logs, templates, prompts, or git history (enforced by Gitleaks in CI).
- Jinja autoescape on; never mark user content `|safe`.
- SQLAlchemy parameterised queries only (no string-built SQL) — CodeQL checks this too.
- Container runs as non-root. Dependencies pinned.
- **No authentication** (assumption; see PRD). "actor" in events is `user`/`ai`/`system`. Listed under known limitations.

---

## 14. DevOps: GitHub Actions, security, EC2, Ansible and monitoring

### 14.1 Required end-to-end deployment flow

1. A push or pull request starts GitHub Actions.
2. Independent quality/security jobs run: Gitleaks (full history), CodeQL (Python), Ruff + pytest, and pip-audit.
3. On a push to the deploy branch (`main`) after required checks pass, the workflow builds the app Docker image on the GitHub-hosted Ubuntu runner. Tag it with the full `${{ github.sha }}`. Do not push the image to a registry.
4. Trivy scans that exact local image. A configured HIGH/CRITICAL failure blocks deployment unless a documented human-approved exception exists.
5. Export the scanned image with `docker save`, compress the TAR if useful, and transfer it directly to EC2 using SCP over SSH. The image is not sent to ECR, Docker Hub, S3 or any registry.
6. Invoke the Ansible deployment playbook from the workflow. Ansible loads the transferred image, writes/updates the deployment `.env` securely, copies Compose and service configuration, and runs `docker compose up -d`.
7. Poll `GET $APP_URL/health` until `commit` equals `${{ github.sha }}` or fail with diagnostics. The app's `/health` must continue to report DB health and the commit SHA.
8. Run OWASP ZAP **baseline/passive** scan against the deployed `APP_URL` after health succeeds. Save the report as a workflow artifact. Do not run destructive active scans against production.

```text
push / pull request
  ├─ Gitleaks ───────────────────────────────┐
  ├─ CodeQL ─────────────────────────────────┤
  ├─ Ruff + pytest ──────────────────────────┤
  └─ pip-audit ──────────────────────────────┘
                 all required checks pass (main push)
                               |
                               v
                   Docker build (tag = commit SHA)
                               |
                         Trivy image scan
                               |
                     docker save -> TAR.gz
                               |
                         SCP over SSH
                               |
                       Ansible deploy
                               |
                    Docker Compose on EC2
                               |
                  /health commit == Git SHA
                               |
                     OWASP ZAP baseline
```

### 14.2 Docker Compose service contract

`compose.yaml` defines exactly these runtime services unless a concrete existing requirement proves otherwise:

| Service | Image/build | Network/ports | Persistent data |
|---|---|---|---|
| `app` | CI-built `loopclose:${IMAGE_TAG}` | Internal `8000`; `/health` and `/metrics`; no public host port | No application data volume required beyond logs if the app writes files. |
| `db` | Official PostgreSQL image, major version selected to match tested local/production compatibility | Internal `5432` only; no public host port | Named `postgres_data` volume. |
| `nginx` | Official Nginx image | Public host `80:80`; optional `443:443` only when TLS configured | Mount Nginx config; certificate volume only if used. |
| `prometheus` | Official Prometheus image | Scrape `app:8000/metrics`; host binding `127.0.0.1:9090:9090` | Named `prometheus_data`; retention default `7d`. |
| `grafana` | Official Grafana image | Host binding `127.0.0.1:3000:3000` | Named `grafana_data`; provisioned Prometheus datasource and dashboard. |

Compose health checks: PostgreSQL uses `pg_isready`; app uses `/health`; dependent services should wait for healthy dependencies where Compose supports it. Use `restart: unless-stopped`. Use a dedicated private network. Never publish database or app ports publicly.

### 14.3 Metrics and monitoring (no exporters)

- Instrument FastAPI with a small Prometheus instrumentation library only if `/metrics` does not already exist. Confirm the library is compatible with the installed FastAPI/Starlette versions and pin the exact version.
- Expose `/metrics` from the app container. Prometheus scrape config uses job `loopclose-app`, target `app:8000`, path `/metrics`, and a modest scrape interval (e.g. 15 seconds).
- Collect application metrics only: request count/rate, request duration/latency histogram, and HTTP response status codes. Avoid high-cardinality labels (raw URL paths, IDs, user input) and never include secrets or request bodies.
- Grafana is provisioned with a Prometheus datasource at `http://prometheus:9090` and one dashboard showing request rate, latency percentiles/histogram and response status/error counts. Keep the dashboard useful and minimal.
- **Do not install or configure Node Exporter, cAdvisor, PostgreSQL Exporter or any other exporter.** Host CPU/RAM/disk and PostgreSQL internals are not part of the requested monitoring scope.
- Prometheus and Grafana web interfaces bind to EC2 loopback. Use SSH forwarding from Windows PowerShell:

```powershell
ssh -i .\ec2-key.pem -L 3000:127.0.0.1:3000 -L 9090:127.0.0.1:9090 ubuntu@EC2_PUBLIC_IP
```

Then open `http://localhost:3000` (Grafana) or `http://localhost:9090` (Prometheus). Do not expose ports 3000/9090 to `0.0.0.0/0`.

### 14.4 Ansible layout and responsibilities

Expected layout (adjust only after inspecting the actual repository):

```text
ops/ansible/
  ansible.cfg
  inventory.ini                 # no secrets; EC2 host supplied by workflow/env
  playbooks/
    provision.yml               # Ubuntu prerequisites, Docker Engine + Compose plugin
    deploy.yml                  # receive already-SCPed image, docker load, compose up, health check
    site.yml                    # orchestrates provision/deploy where appropriate
  templates/
    compose.env.j2              # server-side environment template; never render secrets to logs
  files/
    nginx/default.conf
    prometheus/prometheus.yml
    grafana/provisioning/datasources/prometheus.yml
    grafana/provisioning/dashboards/dashboard.yml
    grafana/dashboards/loopclose-app.json
```

- Provisioning installs Docker Engine and Docker Compose plugin from the official supported repository for the selected Ubuntu release, creates the deployment directory (default `/opt/loopclose`), and configures permissions.
- Deployment copies Compose/Nginx/Prometheus/Grafana config, securely creates `.env` with mode `0600`, loads the transferred image TAR, updates `IMAGE_TAG`, starts/updates the stack and checks `/health`.
- Playbooks must be idempotent, use `become` only where needed, and set `no_log: true` for secret-bearing tasks. Do not print the `.env` contents.
- The GitHub Actions Ubuntu runner may install Ansible for the deployment job. If the human instead runs Ansible from Windows, use WSL2 Ubuntu; native Windows is not the Ansible control-node target.
- `ansible-playbook --syntax-check` must pass. Do not claim provisioning succeeded until it has run against the real EC2 instance.

### 14.5 GitHub Actions secrets and SSH requirements

Create repository or environment secrets using names only (never commit values):

- `EC2_HOST`: EC2 public IP or DNS name.
- `EC2_USER`: Ubuntu AMI default user (commonly `ubuntu`; verify chosen AMI).
- `EC2_SSH_PRIVATE_KEY`: private key matching the EC2 authorized key.
- `EC2_KNOWN_HOSTS`: verified host key entry for the EC2 host. Obtain and verify the fingerprint out-of-band; do not disable host verification.
- `EC2_DEPLOY_PATH`: e.g. `/opt/loopclose`.
- `APP_URL`: public application base URL used by health check and ZAP.
- `DB_PASSWORD`, `GRAFANA_ADMIN_PASSWORD`, `GEMINI_API_KEY`, `GEMINI_MODEL`: sensitive deployment configuration as applicable.

**SSH ingress constraint:** GitHub-hosted runner egress IPs are not a single fixed address. Do not solve this by silently opening EC2 port 22 to the whole internet. The human must choose and approve one approach: (a) a runner with stable egress and a narrow Security Group allowlist; (b) a carefully scoped temporary allowlist workflow with narrowly scoped AWS permissions and guaranteed cleanup; or (c) another approved restricted network path. Document the selected approach and its secrets/permissions in `memory.md`. Keep SSH key authentication and strict host-key verification enabled.

### 14.6 GitHub Actions job design

- Trigger on pull requests and pushes. Run Gitleaks, CodeQL, tests and pip-audit on PRs/pushes. Deployment only on a push to `main` after all required jobs pass.
- Use least-privilege `permissions`; `security-events: write` only for CodeQL upload. Use `fetch-depth: 0` for Gitleaks full history.
- Docker job: checkout, set up Buildx, build/load local image tagged with full SHA, run Trivy against that tag, export with `docker save`, gzip, and retain the TAR only for the deployment job. Do not configure `push: true` or registry login.
- Deploy job: install Ansible, set up SSH key and known_hosts securely, SCP image TAR, run `ansible-playbook` with the tag and secret environment, poll health/commit, then run ZAP baseline. Clean up temporary image TARs and SSH key material even on failure.
- Use concurrency to prevent two deploys to `main` racing. Do not deploy pull-request builds.
- Pin action versions to verified supported tags/SHAs. Check official action repositories before selecting versions.
- CodeQL availability depends on repository visibility and GitHub plan. If unavailable, report it and request the human's choice; do not silently remove it or claim it ran.

### 14.7 Container image and server security

- Dockerfile uses `python:3.12-slim`, installs pinned production dependencies, runs as non-root, exposes only the internal app port, and retains `/health` health check if compatible.
- `.dockerignore` excludes `.git`, `.env*` except no env file is copied, logs, `*.db`, virtualenvs, caches, tests if not needed at runtime, and local secrets.
- PostgreSQL credentials and Grafana admin credentials exist only in the EC2 `.env` with restrictive permissions. The database volume is persistent; never run `docker compose down -v` in deployment.
- Nginx proxies public traffic to `app:8000`; preserve forwarded headers and configure request size/timeouts conservatively. TLS is a follow-up only when a domain and certificate are available.
- EC2 Security Group: public 80 (and 443 only if TLS); SSH restricted by the approved source strategy; no public 3000, 5432, 8000 or 9090.

### 14.8 Health, rollback and operational behaviour

- `/health` checks the database with `SELECT 1` and returns the deployed commit from `GIT_SHA`. AI configuration missing must not fail health if the app's documented fallback behaviour remains available.
- Deployment waits for the app health check and verifies the commit SHA. If the new container fails health, deployment must fail visibly and retain enough previous image information for manual rollback. Do not delete old images automatically until the new version is confirmed healthy.
- PostgreSQL volume persists across app deployments. A container restart is not a database backup; document the single-host/data-loss limitation. Do not introduce S3 as image transport.
- `docker compose logs --tail=...` is the primary operational log inspection path. Existing JSON application logs remain intact.

### 14.9 Branching and commits

`main` is the deployment branch. Use focused conventional commits (`feat:`, `fix:`, `chore:`, `ci:`, `docs:`, `test:`). Run Ruff and pytest before pushing. Never force-push or rewrite history unless the human explicitly asks (the initial fresh-repository setup is a separate human-authorised operation).

## 15. Testing strategy

- `tests/conftest.py`: temp SQLite DB per test session, `TestClient`, fixtures forcing `LLM_PROVIDER=fake`.
- **Must-have tests:** meeting lifecycle transitions (valid/invalid); action item close requires outcome; blocked requires reason; reassign writes event; carry-forward moves open items, increments `carried_count`, writes events, leaves history intact; recap parsing happy path + invalid-JSON retry + fallback heuristic; owner resolution (exact/fuzzy/none); `find_related` fallback chain; `/health`; request-ID header present; log redaction masks secrets.
- No test may call a real LLM or the network.

---

## 16. EC2 resource and performance constraints

- All five services share one EC2 instance. Choose an instance with enough RAM/disk for FastAPI, PostgreSQL, Nginx, Prometheus and Grafana; a 1 GiB instance may be too constrained. Prefer at least 2 GiB RAM if budget/eligibility allows, and confirm current AWS pricing/free-tier eligibility before launch.
- Do not promise free hosting or high availability. AWS EC2, EBS storage, data transfer, public IPv4 and snapshots may incur charges depending on account/region/current offers.
- Keep one Uvicorn worker initially; no local ML models. AI calls remain outbound to Gemini and use existing timeouts/fallbacks.
- Prometheus retention defaults to 7 days and must be bounded. Monitor disk use manually; no Node Exporter/cAdvisor is in scope.
- PostgreSQL data is persisted in a named Docker volume. A single EC2 volume is still a single failure domain; establish a backup procedure before treating the data as durable.
- The app must remain usable when AI is unavailable; metrics instrumentation must not block request handling.

## 17. Known limitations

- Single EC2 host: no high availability or automatic failover; host/disk failure can take down app, DB and monitoring.
- PostgreSQL is self-managed in Docker on the same EC2 instance; patching, backups, disk capacity and recovery are the project owner's responsibility.
- No authentication/authorisation in the application; the existing “Viewing as” selector is not a security boundary. Do not expose sensitive real data.
- No email/calendar/chat integrations; content is manually entered or pasted; no audio transcription.
- No Alembic migrations; existing `create_all()` approach is time-boxed and has schema-change limitations.
- No Nudge Center, Level B vector RAG, embeddings or Ask Team Memory (explicitly deferred).
- Prometheus/Grafana observe application metrics only. No host, container or PostgreSQL exporters are deployed.
- Monitoring UIs are loopback-bound and require an SSH tunnel by default.
- Direct SCP deployment requires a carefully restricted SSH ingress strategy for GitHub Actions runners.
- AI quality/rate limits depend on the configured Gemini model and account.
- TLS requires a domain and certificate configuration; HTTP may be used for an initial controlled demo only.

## 18. Source-of-truth precedence

If documents disagree: `PRD.md` (behaviour) > `ARCHITECTURE.md` (tech/structure) > `design.md` (visuals) > `TASKS.md` (planning). **Do not silently pick**: choose the most conservative option, proceed, and record the conflict under "Open Questions" in `memory.md` for the human.
