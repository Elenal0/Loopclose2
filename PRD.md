# PRD.md — LoopClose (Product Requirements Document)

> Defines **what** is being built and **how it must behave**. Tech choices live in `ARCHITECTURE.md`, visuals in `design.md`.
> Origin: SPIT / DigiPlus Technical Assessment — "AI-Powered Knowledge & Tasks Management Platform" (30 Oct 2026, 3.5 hours).

---

## 1. Problem statement

Teams lose context every time work changes hands. Someone misses a meeting or never responds to the invite; a colleague goes on leave; an action item from last week never reaches this week's agenda. Information is scattered across documents, personal notes, chats, meeting recordings and emails.

**LoopClose** captures what was discussed and decided, transfers that context to the right people, and follows up on open items until they are closed — with AI doing the tedious structuring, summarising and linking.

## 2. Goals and non-goals

### Goals
1. Create and persist **meetings** and **work handovers** (agenda, participants, notes, decisions, action items).
2. Manage their **lifecycle** and track every **action item** (owner, due date, status, outcome) until closed.
3. **Carry open items forward** to the next meeting/handover while keeping their history.
4. Use AI to turn **raw unstructured notes** into a structured recap (summary, decisions, action items, owners, due dates).
5. Generate a short **catch-up brief** for anyone who missed a meeting or is taking over.
6. Maintain a small **knowledge base** and use AI (RAG) to **link** items to it.
7. **Record the outcome** of each item.
8. Provide **validation and error handling**, **structured logging**, and **graceful AI-failure handling**.
9. Be deployed to one AWS EC2 instance through an automated, security-scanned GitHub Actions pipeline.
10. Provide application-level monitoring with Prometheus and Grafana.

### Non-goals (do not build)
Authentication/user accounts; roles/permissions; email, calendar, Slack or Teams integrations; audio/video transcription (users paste transcripts); file parsing beyond plain text/markdown paste/upload; real-time collaboration; mobile app; multi-tenancy; payments; Nudge Center; Level B vector RAG/embeddings/chunking; Ask Team Memory; managed database services; container registries.

## 3. Users and personas

| Persona | Situation | What they need |
|---|---|---|
| **Aarav — Team Lead** | Runs weekly syncs, hates chasing people | Fast recap from messy notes, clear owners/dates, see what is overdue |
| **Sneha — Absent teammate** | Missed Tuesday's meeting | A 60-second brief: what was decided, what's on my plate, what changed |
| **Priya — Going on leave** | Needs to hand over her work | Create a handover, reassign her open items, leave notes |
| **Rohan — Taking over** | Inherits Priya's work | Brief on context + her open items + related documents/history |

> There is no login. The UI has a "Viewing as" person selector (stored in a plain cookie, not a security feature) used only to personalise dashboard and briefs.

## 4. Core concepts / glossary

- **Meeting** — a discussion event. **Handover** — a work-transfer record from person A to person B. Same table, `kind` differs; both have agenda, participants, notes, decisions, items.
- **Action item** — a task with owner, due date, priority, status, outcome and a full event history.
- **Origin meeting** — where an item was first raised (never changes). **Current meeting** — whose agenda it's on now (changes on carry-forward).
- **Carry-forward** — moving open items to the next meeting/handover, incrementing `carried_count` and writing history.
- **Recap** — AI-produced structured proposal (summary, decisions, items) from raw notes. Human reviews and applies.
- **Catch-up brief** — AI-produced short document for one person about one meeting/handover.
- **Knowledge base (KB)** — documents (notes, docs, chats, emails, transcripts) plus auto-indexed past meetings.
- **Link** — an association between an item/decision/meeting and a KB document, with a reason and a method (`llm`, `keyword`, `manual`).

## 5. Functional requirements

Priority: **P0** = must ship, **P1** = should ship, **P2** = optional stretch.

### 5.1 People
- **FR-P1 (P0)** Create and list people (name required, email/role optional). Mark a person inactive (e.g. on leave).
- **FR-P2 (P0)** "Viewing as" selector in the header.

### 5.2 Meetings and handovers
- **FR-M1 (P0)** Create a meeting or handover with title, kind, scheduled date/time, agenda, participants.
- **FR-M2 (P0)** Handover requires `from` and `to` people (different); both auto-added as participants.
- **FR-M3 (P0)** List meetings with filters (kind, status); show counts of open items.
- **FR-M4 (P0)** Meeting detail page with tabs: Overview, Notes & Recap, Items, Brief, Knowledge, History.
- **FR-M5 (P0)** Lifecycle: `planned → held → closed`, `planned|held → cancelled`. Invalid transitions rejected with a clear message.
- **FR-M6 (P0)** Track **attendance** per participant: invited, attended, absent, no_response, declined (covers "never responded to the invite").
- **FR-M7 (P0)** Save raw notes (paste meeting notes, chat export, transcript).
- **FR-M8 (P0)** Add/delete decisions manually.
- **FR-M9 (P1)** Create a new meeting from a previous one (`previous_meeting_id`) with a "Carry forward N open items" checkbox (default on).
- **FR-M10 (P0)** Handover: one-click "Reassign all of <from>'s open items to <to>" (writes `reassigned` events).

### 5.3 Action items
- **FR-A1 (P0)** Create item: title, description, owner, due date, priority; belongs to a meeting.
- **FR-A2 (P0)** Status changes: open, in_progress, blocked (needs reason), reopen.
- **FR-A3 (P0)** **Close with outcome**: outcome type (completed / partially_completed / dropped / superseded) + outcome note (≥5 chars). Only way to reach done/dropped.
- **FR-A4 (P0)** Reassign owner; due date edits recorded.
- **FR-A5 (P0)** **History timeline** per item showing every event with time, actor, from→to values and meeting context.
- **FR-A6 (P0)** Global items list with filters: status, owner, overdue, meeting; sort by due date.
- **FR-A7 (P0)** Overdue and unassigned items visibly flagged everywhere they appear.

### 5.4 Carry-forward
- **FR-C1 (P0)** `POST carry-forward` moves all open/in_progress/blocked items from meeting X to meeting Y: updates `current_meeting_id`, increments `carried_count`, writes `carried_forward` event. Done/dropped items are never moved.
- **FR-C2 (P0)** The destination meeting shows a **"Carried over"** section at the top listing each item with its origin meeting, carried count and age.
- **FR-C3 (P0)** Closing a meeting with open items prompts: "Carry N open items to…" (select next meeting or create one).

### 5.5 AI recap
- **FR-R1 (P0)** From raw notes, "Generate recap" produces: summary (≤120 words), decisions, action items (title, owner, due date, priority), open questions.
- **FR-R2 (P0)** Owners are matched to participants (exact then fuzzy ≥0.8); unmatched → "needs owner".
- **FR-R3 (P0)** Relative dates ("by Friday", "next Tuesday") are resolved relative to the meeting date.
- **FR-R4 (P0)** The result is shown as a **review form** (checkbox per row, editable text); "Apply selected" creates decisions/items (`source=ai`) and stores the summary.
- **FR-R5 (P0)** On AI failure: retry once automatically; then use **heuristic fallback** and show a banner with **Retry**. Raw notes are never lost. See `ARCHITECTURE.md` §10.3.
- **FR-R6 (P0)** AI-created rows carry an "AI" badge until edited by a human.

### 5.6 Catch-up brief
- **FR-B1 (P0)** For a given meeting/handover and person, generate a brief with: TL;DR, key decisions, "your items", what changed since, open questions, related knowledge, suggested first steps.
- **FR-B2 (P0)** Default person: anyone with attendance `absent`/`no_response`, or the handover `to` person; any person can be chosen.
- **FR-B3 (P0)** Briefs are saved (`catchup_briefs`) and can be regenerated. Fallback = template brief from DB data.
- **FR-B4 (P1)** "Copy brief" button.

### 5.7 Knowledge base and RAG
- **FR-K1 (P0)** Add a document (title, type, pasted/uploaded `.txt`/`.md` content). On create: AI summary + tags (fallback: trimmed text).
- **FR-K2 (P0)** Past meetings are auto-added/refreshed as KB documents (type `meeting`) when a recap is applied or a meeting is closed.
- **FR-K3 (P0)** "Suggest related knowledge" on an item/decision/meeting: returns up to 3 matches, each with document title, relevance, reason and method; user can accept (store link) or dismiss; manual add/remove of links is possible.
- **FR-K4 (P0)** Retrieval goes through `find_related()`; Level A uses LLM catalogue matching with keyword fallback (see `ARCHITECTURE.md` §11).
- **FR-K5 (P0)** Linked knowledge is shown on item detail, meeting "Knowledge" tab, and inside the catch-up brief.

> **Scope note:** Level B vector RAG and Ask Team Memory are explicitly deferred. Preserve the implemented Unit 6 Level A linking behaviour; do not add embedding/chunking code.

### 5.8 Dashboard
- **FR-D1 (P0)** Cards: My open items (for the "viewing as" person), Overdue, Upcoming meetings (next 7 days), Recent decisions, and "Needs owner" count.

## 6. AI requirements (product view)

1. AI is a **meaningful** part of the solution: recap, brief, KB summarisation and Level A knowledge linking.
2. The app **must remain fully usable without AI** (manual entry everywhere; fallbacks for every AI operation).
3. AI output is **validated** against schemas; invalid output triggers retry then fallback.
4. AI suggestions are **proposals** the user reviews, not silent writes (recap → review → apply; links → accept/dismiss).
5. Every AI call is logged with latency and outcome (no content, no secrets).
6. Configuration (provider, model, timeouts) is by environment variables and documented in README.

## 7. Logging requirements (product view)

Structured JSON logs to file and stdout; levels; a correlation/request ID on every record and response header; key business events (created, reassigned, closed, carried forward); AI calls with latency/outcome; errors with context; no secrets. Full spec: `ARCHITECTURE.md` §12.

## 8. Validation rules

| Field / action | Rule | Error message (example) |
|---|---|---|
| Person name | required, 1–100 chars | "Name is required." |
| Person email | optional, valid format, unique | "That email is already used." |
| Meeting title | 3–200 chars | "Title must be at least 3 characters." |
| Scheduled at | required, valid datetime | "Pick a date and time." |
| Agenda | ≤ 5,000 chars | "Agenda is too long (max 5,000)." |
| Raw notes | ≤ 50,000 chars; ≥ 20 chars to run recap | "Add at least a few sentences of notes first." |
| Participants (meeting) | ≥ 1 | "Add at least one participant." |
| Handover | from ≠ to, both required | "Choose two different people." |
| Status transition | must follow state machine | "A closed meeting can't be reopened." |
| Action item title | 3–200 chars | "Give the item a short title (3+ characters)." |
| Due date | not in the past **when creating**; (past allowed for existing items → shows overdue) | "Due date can't be in the past." |
| Owner | must exist and be active | "Choose an active person as owner." |
| Blocked | `blocked_reason` required (3–300 chars) | "Say what is blocking this." |
| Close | `outcome_type` + `outcome_note` ≥5 chars | "Record the outcome before closing." |
| KB document | title 3–200; content 20–100,000 chars | "Document content is too short." |
| Uploads | `.txt` / `.md` only, ≤ 1 MB, UTF-8 | "Only .txt or .md files up to 1 MB." |
| Duplicate KB link | unique per (entity, document) | handled silently ("Already linked.") |

Validation errors appear **inline next to fields** (HTML) or in the standard JSON error shape (API); HTTP 422. State errors → 409. Missing records → 404 with friendly page.

## 9. Lifecycles (summary; authoritative diagrams in `ARCHITECTURE.md` §8)

- Meeting: `planned → held → closed`; `planned|held → cancelled`.
- Action item: `open ⇄ in_progress ⇄ blocked → (close with outcome) → done | dropped → (reopen) → open`.

## 10. Demo data requirements ("quality of your own data")

The seed script (`scripts/seed.py`, idempotent) creates **realistic, human-written** data, with dates **relative to today** so overdue/upcoming states appear:

- **6 people** (varied roles; one inactive "on leave").
- **4 meetings** over two weeks: a sprint planning, a vendor sync (a participant `no_response`, another `absent`), a retro, and **1 handover** (Priya → Rohan).
- **~14 action items** across statuses; at least: 2 overdue, 1 blocked, 1 carried 3×, 2 closed with different outcome types, 1 unassigned, 1 reassigned (with history).
- **6 KB documents**: engineering guidelines, release checklist, incident post-mortem, onboarding doc, vendor-contract email thread, a team chat export; plus auto-indexed meeting docs.
- **1 sample raw-notes blob** (messy, mixed-format, with relative dates and mis-spelled names) stored in a planned meeting, ready for a live recap demo.
- A few pre-existing KB links and one saved brief.

## 11. Non-functional requirements

- Core pages should remain responsive on the selected EC2 instance; AI actions show a loading state and complete within the AI timeout.
- Works on desktop and mobile widths (responsive).
- Accessible: keyboard navigable, visible focus, labelled inputs, sufficient contrast (`design.md`).
- No secrets in repo/logs; pipeline blocks leaks and code-scanning findings.
- Runs on one EC2 instance with resources sized for app, PostgreSQL and monitoring containers; no high-availability claim.

## 12. Assessment traceability

| Assessment requirement | Where satisfied |
|---|---|
| Create & persist meetings / work transfers (agenda, participants, notes, decisions, items) | FR-M1–M8, FR-A1 |
| View/manage lifecycle; track every item until closed | FR-M5, FR-A2–A7 |
| Carry open items forward, keep history | FR-C1–C3, `action_item_events` |
| AI: unstructured → structured recap | FR-R1–R6 |
| Catch-up brief for absentees / successors | FR-B1–B4 |
| Knowledge base + AI linking | FR-K1–K6 |
| Record outcome of each item | FR-A3 |
| Validation & error handling | §8, `ARCHITECTURE.md` §13 |
| AI meaningful + graceful failure | §6, `ARCHITECTURE.md` §10.3 |
| Structured logging (levels, correlation ID, key events, AI latency, errors, file, no secrets) | §7, `ARCHITECTURE.md` §12 |
| CI/CD and security pipeline | `ARCHITECTURE.md` §14: Gitleaks, CodeQL, Ruff/pytest, pip-audit, Trivy and OWASP ZAP baseline |
| Cloud deployment | AWS EC2 + Docker Compose + Ansible, `ARCHITECTURE.md` §14 |
| Application monitoring | Prometheus + Grafana, application metrics only, `ARCHITECTURE.md` §14 |

## 13. Assumptions (repeat in README)

1. Meeting "recordings" are provided as pasted transcript text.
2. Scattered sources (docs, notes, chats, emails) are ingested by paste/upload; no live integrations.
3. No authentication; "Viewing as" is a convenience, not security.
4. Single team / single tenant; times stored in UTC, displayed in the browser's locale (default Asia/Kolkata for seed text).
5. AI uses a free-tier model; quality and rate limits may vary.
6. Demo data is synthetic, written by the author.

## 14. Acceptance criteria (final demo, ~5 minutes)

1. Open dashboard → see open items, overdue items, upcoming meetings and needs-owner count.
2. Open the planned meeting with seeded raw notes → **Generate recap** → review → **Apply** → items appear with owners/dates and an AI badge.
3. Disable the AI key (or use fake failure toggle) → recap shows **fallback banner** and heuristic items; notes intact.
4. Close the meeting → prompted to **carry forward** → destination shows "Carried over" section; item history shows `carried_forward`.
5. Open the handover → **reassign** Priya's items to Rohan → history shows `reassigned`.
6. Open **Brief** as Rohan / Sneha → readable brief with related knowledge and sources.
7. Item detail → **Suggest related knowledge** → accept a link → appears on item and in brief.
8. **Close an item** with outcome; try closing without outcome → validation error.
9. Show `logs/app.log` lines for key actions with a shared `request_id`.
10. Show the GitHub Actions run with Gitleaks, CodeQL, tests, pip-audit, Trivy, SCP/Ansible deployment and health check green.
11. Open the EC2-hosted LoopClose URL and demonstrate the Prometheus target as UP and Grafana application dashboard populated with request rate, latency and status-code metrics.
12. Run/show the OWASP ZAP baseline report after deployment.
