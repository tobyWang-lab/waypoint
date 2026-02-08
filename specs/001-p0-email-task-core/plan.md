# Implementation Plan: P0 Email-to-Task Core Pipeline

**Branch**: `001-p0-email-task-core` | **Date**: 2026-02-08 | **Spec**: [specs/001-p0-email-task-core/spec.md](specs/001-p0-email-task-core/spec.md)  
**Input**: Feature specification from `/specs/001-p0-email-task-core/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

---

## Summary

Deliver a modular n8n core pipeline that ingests Gmail, buffers minimal derived data in internal tables (ODS), and delivers **Task Cards to Telegram** for Human-in-the-Loop (HITL) navigation.

**Implemented pipeline sequence (matches current `workflows/` folder):**
1. **Email Ingestion (Buffered)** → `Waypoint-mail-retriever.json`  
   - Fetch unread emails via n8n Gmail node + native OAuth2 credentials  
   - De-duplicate by Gmail `messageId`  
   - Buffer minimal identifiers/derived fields into ODS tables (no raw body retention beyond the active window)
2. **Telegram HITL Approve/Decline (Callback-driven)** → `Waypoint-Telegram-Approve Or Decline Flow.json`  
   - Handles interactive button callbacks  
   - Updates approval state for each task candidate (no external side effects in current scope)
3. **Telegram Intelligence Report / Navigation Hub** → `Waypoint-Collect_TG.json`  
   - Publishes daily/high-level summaries (Tasks / FYI / Noise counts, critical items)  
   - Supports navigation context for review (per spec)

**Explicitly Deferred (not implemented in current repo snapshot):**
- Google Tasks synchronization workflow and field mapping execution (Epic 3)
- Any “Sync Expert” / reconciliation logic with Google Tasks API

---

## Technical Context

**Language/Version**: n8n workflow JSON (modular multi-workflow architecture)

**Primary Dependencies (implemented today)**:
- n8n (self-hosted Docker)
- Gmail API (via n8n Gmail node)
- Telegram Bot API (notifications + HITL callbacks)
- LLM provider (via n8n AI Agent node) for:
  - Noise vs Signal classification (Noise does **not** generate Task Cards)
  - Thread-aware analysis and waypoint extraction into Task Cards

**Primary Dependencies (deferred / not implemented in current `workflows/`)**:
- Google Tasks API (task create/update/sync)

**Storage (Operational Data Store / ODS)**:
- n8n Internal Data Tables used as a temporary buffer to support thread grouping, dedupe, and HITL navigation:
  - `Mail_Table`
  - `LLM_Table`
  - `Task_Table`

**Stateless Enforcement (required by spec)**:
- n8n execution data saving disabled/pruned to avoid raw content retention in execution history
- ODS tables MUST NOT persist raw email body content beyond the active processing window
- TTL/pruning applied; retain only minimal non-PII metadata needed for correlation/audit/idempotency

**Testing Strategy**:
- Workflow-level execution tests for the three shipped workflows
- Schema validation: all generated Task Cards must conform to the JSON Schema in `spec.md` (SC-008)
- Privacy checks: verify no raw email content persists after processing completes (SC-007)

---

## Performance Goals & Constraints (Synced to spec Success Criteria)

**Performance Goals**
- **SC-002**: Process 100 emails and generate Task Cards in under 5 minutes.
- **SC-014**: Telegram notifications delivered within 5 minutes of scheduled trigger with template compliance.

**Constraints**
- HITL-first delivery: Task Cards are delivered to Telegram for review/navigation (Google Tasks sync is deferred).
- Thread awareness: group by `ThreadId` prior to AI analysis (per spec requirements).
- Deduplication: Gmail `messageId` is the primary dedupe key to prevent re-processing.
- Timezone: Asia/Taipei for timestamps and date interpretation.
- Input limits: cap LLM input size; skip oversized items with partial status + reason (no raw body storage).
- Stateless/privacy: no raw email content or PII persisted beyond the processing window (SC-007).

---

## Constitution Check (Re-evaluated vs updated spec)

- **Principle I (Code Quality)**: **PASS**  
  Workflows are separated by responsibility: ingestion, HITL callback handling, and Telegram reporting/hub.

- **Principle II (Testing Standards)**: **PASS**  
  Workflows can be tested independently; Task Card schema validation is explicit (SC-008).

- **Principle III (User Experience)**: **PASS**  
  Telegram summaries + interactive buttons provide the review/approval UX without a custom frontend.

- **Principle IV (MVP & Anti-Over-Design)**: **PASS**  
  Current scope avoids Google Tasks sync and focuses on Telegram HITL + intelligence reporting.

- **Principle V (Stateless Processing)**: **PARTIAL**  
  ODS tables introduce state (by design) for correlation, dedupe, and HITL navigation.  
  However, the spec mandates stateless enforcement via execution pruning + ODS TTL + minimal-field storage.

- **Principle VI (Integration Flexibility)**: **PASS**  
  LLM provider and downstream sync are swappable; Epic 3 is isolated and deferred.

- **Principle VII (Error Handling)**: **PARTIAL**  
  Spec defines “Continue on Fail” + inline validation + Error Trigger strategy, but the current `workflows/` folder does **not** include a dedicated error-trigger workflow JSON artifact.  
  Plan action: ensure failures record non-PII metadata + statuses in ODS, and add an Error Trigger workflow when introducing broader retries/alerting requirements.

---

## Project Structure

### Documentation

```text
specs/001-p0-email-task-core/
├── plan.md              # This file
├── spec.md              # Feature specification
└── tasks.md             # Task list and progress tracking
```

### Source Code (as currently present)

```text
workflows/
├── Waypoint-mail-retriever.json                    # Email ingestion (buffered)
├── Waypoint-Telegram-Approve Or Decline Flow.json  # HITL callback handling (Approve/Decline)
└── Waypoint-Collect_TG.json                        # Telegram intelligence report / navigation hub
```

### Deferred Workflow Artifacts (not present in repo snapshot)

```text
workflows/
├── Waypoint-task.json                 # Google Tasks sync / reconciliation (Epic 3)
└── waypoint-error-trigger.json        # Centralized error trigger / alerting (recommended)
```

---

## Success Criteria Traceability (Plan → Spec)

**In-scope (implemented now):**
- **SC-002**: Achieved by efficient ingestion + bounded LLM calls + buffered ODS processing
- **SC-007**: Enforced via execution pruning + minimal ODS fields + TTL/pruning
- **SC-008**: Enforced via Task Card JSON Schema validation step
- **SC-014**: Achieved by `Waypoint-Collect_TG.json` Telegram delivery design

**Deferred (requires Epic 3 workflows):**
- **SC-011 / SC-012 / SC-013**: Google Tasks sync outcomes

---

## Data Schema Alignment (ODS Tables + Task Card)

**Key alignment rules from spec**
- ODS stores **minimal identifiers/derived fields**, not raw email bodies (beyond active processing window)
- Dedupe key: Gmail `messageId` (and optionally `threadId`)
- Thread awareness: grouping by `ThreadId` before AI analysis

### `Mail_Table` (buffer)
- MUST include: `messageId`, `threadId`, timestamps (Asia/Taipei), Gmail deep link (or fields needed to build it)
- SHOULD include: snippet/derived fields only if allowed by privacy requirement
- MUST NOT retain full raw body beyond active window

### `LLM_Table` (derived intelligence)
- Stores classification outputs, model/prompt metadata, status, and non-PII error metadata
- Supports partial results when oversized/failed (with statusReason)

### `Task_Table` (Task Cards + HITL status)
- Stores Task Cards that conform to JSON Schema in spec:
  - `title`, `sourceLink`, `intent` (Noise excluded from TaskCard), `dueDate` (ISO date or null), plus required audit/status fields
- Stores HITL state transitions (PENDING/APPROVED/DECLINED) used by the Approve/Decline workflow

---

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| Principle V (Stateless Processing) | ODS buffering is required for thread grouping, dedupe, and HITL navigation | Pure stateless execution cannot support reliable correlation and user-driven approvals across runs |
| Principle VII (Error Handling) | Spec requires robust handling without retaining raw content | Pure node-level failure handling is insufficient for auditability and consistent partial-status reporting |