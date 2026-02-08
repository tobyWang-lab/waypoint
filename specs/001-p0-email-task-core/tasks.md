---

description: "Task list for P0 Email-to-Task Core Pipeline"
---

# Tasks: P0 Email-to-Task Core Pipeline

**Input**: `/specs/001-p0-email-task-core/plan.md` and `/specs/001-p0-email-task-core/spec.md`  
**Prerequisites**: `plan.md` (required), `spec.md` (required), `quickstart.md` (recommended), `testing.md` (recommended), `compliance.md` (recommended)

## Scope Anchor (MUST match plan.md + repo snapshot)

**Workflows present in `/workflows/` (in-scope):**
- `workflows/Waypoint-mail-retriever.json`
- `workflows/Waypoint-Telegram-Approve Or Decline Flow.json`
- `workflows/Waypoint-Collect_TG.json`

**Explicitly deferred (out of scope for this repo snapshot):**
- Google Tasks sync workflow (Epic 3) and any “Sync Expert” logic
- Dedicated Error Trigger workflow JSON artifact (`workflows/waypoint-error-trigger.json`)

**Implemented execution sequence (as per plan.md):**  
`Waypoint-mail-retriever.json` → `Waypoint-Telegram-Approve Or Decline Flow.json` → `Waypoint-Collect_TG.json`

---

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1–US4)
- Include exact file paths in descriptions

---

## User Stories (Plan-aligned)

- **US1 — Ingestion & Buffering**: Fetch unread Gmail, dedupe by `messageId`, buffer minimal derived data in ODS.
- **US2 — Telegram HITL**: Handle Approve/Decline callbacks and persist approval state (no external side effects).
- **US3 — Telegram Intelligence Report**: Publish summaries (Tasks/FYI/Noise counts) and navigation context.
- **US4 — Compliance & Validation**: Stateless enforcement (SC-007), schema validation (SC-008), performance + delivery checks (SC-002/SC-014).

> Note: AI classifier/extraction is referenced in spec/plan as a dependency, but there is **no separate classifier workflow JSON** in `/workflows/` today. Any AI steps must exist inside the current three workflows, or remain deferred.

---

## Phase 1: Environment & Stateless Enforcement (Setup)

**Goal**: Ensure the runtime is configured to avoid raw content retention and to support audits.

- [ ] T001 [P] Confirm execution data retention is disabled/pruned (SC-007) and document exact settings in `specs/001-p0-email-task-core/quickstart.md`.  
  DoD: quickstart lists the exact n8n settings/env vars to enforce pruning; Verify: reviewer can apply settings and see executions do not retain raw email content.

- [ ] T002 [P] Create/update privacy audit checklist (SC-007) in `specs/001-p0-email-task-core/compliance.md`.  
  DoD: checklist verifies (1) execution history, (2) ODS tables, (3) logs contain no raw email body/PII beyond allowed minimal fields; Verify: checklist is runnable.

- [ ] T003 [P] Create workflow test checklist (SC-002/SC-008/SC-014) in `specs/001-p0-email-task-core/testing.md`.  
  DoD: checklist covers happy path + partial/failure cases for the three workflows; Verify: checklist is runnable.

---

## Phase 2: Email Ingestion (Buffered) — US1

**File**: `workflows/Waypoint-mail-retriever.json`

- [ ] T010 [US1] Verify Gmail ingestion filters (unread inbox) and excludes already-processed duplicates per spec/plan.  
  DoD: ingestion only returns intended messages; Verify: run workflow against a test mailbox.

- [ ] T011 [US1] Implement/verify deduplication by Gmail `messageId` before writing to ODS.  
  DoD: re-running ingestion does not create duplicates; Verify: run twice and compare ODS rows.

- [ ] T012 [US1] Enforce input-size cap and partial status for oversized items (no raw body retention).  
  DoD: oversize emails are marked partial/skipped with non-PII `statusReason`; Verify: test with oversized content.

- [ ] T013 [US1] Ensure ODS writes contain **minimal identifiers/derived fields only** (SC-007).  
  DoD: no full raw body persisted beyond active window; Verify: inspect Data Table write payloads.

- [ ] T014 [US1] Add/verify stable correlation identifiers that can be used by Telegram HITL callbacks (e.g., `messageId`, `threadId`, taskCandidateId).  
  DoD: downstream callback can uniquely map back to the correct candidate; Verify: end-to-end click test with a known message.

---

## Phase 3: Telegram HITL Approve/Decline — US2

**File**: `workflows/Waypoint-Telegram-Approve Or Decline Flow.json`

- [ ] T020 [US2] Verify Telegram callback trigger is configured and reliably receives button presses.  
  DoD: Approve/Decline button press triggers workflow; Verify: manual Telegram test.

- [ ] T021 [US2] Implement/verify approval state transitions persisted in ODS (e.g., `Task_Table`).  
  DoD: states include at least PENDING/APPROVED/DECLINED with timestamps; Verify: inspect ODS updates.

- [ ] T022 [US2] Confirm **no external side effects** occur on approval/decline in current scope (Google Tasks sync is deferred).  
  DoD: no Google Tasks API calls exist; Verify: node inventory review.

- [ ] T023 [US2] Validate callback payload integrity (signature/secret if used; otherwise tokenized callback_data) and reject malformed/unmapped callbacks.  
  DoD: invalid callbacks do not mutate state; Verify: send malformed callback test.

---

## Phase 4: Telegram Intelligence Report / Navigation Hub — US3

**File**: `workflows/Waypoint-Collect_TG.json`

- [ ] T030 [US3] Implement/verify report aggregates: Tasks / FYI / Noise counts + critical items (SC-014).  
  DoD: message includes required sections and counts; Verify: run report after ingestion.

- [ ] T031 [US3] Ensure the report supports navigation context (links/IDs) consistent with HITL workflow correlation keys.  
  DoD: items in report map to actionable HITL buttons/records; Verify: click-through + callback updates intended record.

- [ ] T032 [US3] Validate Telegram formatting stability (Markdown/MarkdownV2 as configured) to avoid message send failures.  
  DoD: no formatting exceptions across typical content; Verify: run with edge-case characters.

---

## Phase 5: Schema Validation, Performance, and Privacy — US4

**Goal**: Meet the plan’s in-scope Success Criteria: SC-002, SC-007, SC-008, SC-014.

- [ ] T040 [US4] Add/verify Task Card schema validation step (SC-008) within the workflow(s) that produce Task Cards.  
  Files: `workflows/Waypoint-mail-retriever.json` and/or `workflows/Waypoint-Collect_TG.json` (wherever Task Cards are generated).  
  DoD: invalid cards are rejected or marked partial with non-PII reason; Verify: inject invalid sample.

- [ ] T041 [US4] Run privacy audit (SC-007) using `specs/001-p0-email-task-core/compliance.md`.  
  DoD: audit passes; Verify: attach evidence notes (what was checked, where).

- [ ] T042 [US4] Run performance check (SC-002) on a 100-email batch using the test checklist.  
  DoD: under 5 minutes end-to-end (or within the target in spec); Verify: record timestamps and environment details.

- [ ] T043 [US4] Verify Telegram delivery timing + template compliance (SC-014).  
  DoD: scheduled/manual runs deliver within 5 minutes; Verify: capture send timestamps and rendered message.

---

## Deferred / Out of Scope (Epic 3)

> These tasks are intentionally deferred until corresponding workflow JSON files exist under `/workflows/`.

- [ ] D001 Implement Google Tasks sync workflow JSON (Epic 3), including field mapping and RFC 3339 conversion.
- [ ] D002 Implement “Sync Expert” reconciliation logic (Create/Update/Skip) + idempotency keys.
- [ ] D003 Add dedicated error-trigger workflow JSON artifact for centralized alerting/auditability.

---

## Definition of Done (Overall)

- The three in-scope workflows execute in the documented sequence and interoperate via stable correlation keys.
- SC-002 / SC-007 / SC-008 / SC-014 are demonstrably testable and pass via `testing.md` + `compliance.md`.
- ODS usage is minimal and compliant: no raw email body retention beyond the active window; execution history is pruned/disabled.
- No Google Tasks side effects exist in the current implementation (explicitly deferred).