# Feature Specification: P0 Email-to-Task Core Pipeline

**Feature Branch**: `001-p0-email-task-core`  
**Created**: 2026-01-18  
**Status**: Draft  
**Input**: P0 Email-to-Task Core Pipeline covering Signal Acquisition (Epic 1) and Intelligence Navigation (Epic 2)

---

## Overview

This specification defines the P0 (Critical) features required to build the core "Email-to-Task" pipeline for WAYPOINT.

**Implemented workflow set (source-of-truth = repository `workflows/` folder):**
- `Waypoint-mail-retriever.json`
- `Waypoint-Telegram-Approve Or Decline Flow.json`
- `Waypoint-Collect_TG.json`

The **core pipeline sequence (as implemented)** is:

1. **Email Ingestion (Buffered)** → `Waypoint-mail-retriever.json`  
2. **Telegram HITL Approval/Decline (Webhook callback flow)** → `Waypoint-Telegram-Approve Or Decline Flow.json`  
3. **Telegram Intelligence Report / Navigation Hub** → `Waypoint-Collect_TG.json`

> Note: The Approve/Decline workflow is event-driven (Telegram callback). It must be deployed and active; it is part of the end-to-end sequence even though it is triggered asynchronously by user interaction.

**Scope alignment note (important):**
- This repository snapshot does **not** include a Google Tasks synchronization workflow JSON (e.g., `Waypoint-task.json`).  
  Therefore, **Epic 3 (Synchronization)** and any “Google Tasks state sync” requirements are treated as **Deferred / Not implemented in current workflows folder** in this spec revision.

All features adhere to the Constitution's **Stateless Processing** principle—no raw email content persists beyond processing (details in “Deployment & Stateless Processing”).

---

## Target Environment: n8n

This specification defines the Email-to-Task pipeline as a modular n8n workflow architecture. All components MUST:
- Be deployable as n8n workflow nodes or steps
- Use **n8n native nodes** for all interactions (Gmail, LLM via AI Agent/HTTP, Telegram)
- Use n8n's HTTP request capabilities for external service integration
- Leverage n8n's credential management for OAuth and API authentication
- Be exported and versioned in n8n's JSON workflow format
- Be testable within n8n's workflow execution environment

### Gmail Integration

- Use n8n's **built-in Gmail node** with native OAuth2 credentials
- Token refresh is handled automatically by n8n's credential system
- No custom OAuth2 flow implementation required
- (If/when Google Tasks sync is added) OAuth credential should include Google Tasks scope (`https://www.googleapis.com/auth/tasks`) for unified authorization

### Human-in-the-Loop (HITL) Navigation

- Implement **Interactive Telegram Buttons** (Approve/Decline) for each extracted task candidate.
- Approval decisions are handled by the dedicated workflow: **`Waypoint-Telegram-Approve Or Decline Flow.json`**
- Downstream “Intelligence Report / Navigation Hub” is handled by: **`Waypoint-Collect_TG.json`**
- Any action that would create external side effects (e.g., Google Tasks writes) MUST be gated behind explicit approval (when Epic 3 is implemented).

### Noise Filtering Implementation

- Use n8n's **AI Agent node** with a lightweight LLM call for noise classification
- LLM determines Noise vs Signal and categorizes noise type (Advertisement, Newsletter, Notification)
- Noise results should not generate a Task Card; they should be counted and summarized in Telegram reports

### Deployment & Stateless Processing

- Deploy as **self-hosted n8n via Docker**
- Environment-agnostic and container-ready per **Constitution v2.0.0**
- **Buffered Data Architecture (ODS)**: Uses internal n8n Data Tables (e.g., `Mail_Table`, `LLM_Table`, `Task_Table`) as a temporary Operational Data Store (ODS) to enable thread-level grouping and HITL navigation.
- **Stateless enforcement requirement**:
  - Execution data saving is disabled / pruned so raw content is not retained in n8n execution history.
  - ODS tables MUST NOT persist raw email body content beyond the active processing window. Store only minimal identifiers/derived fields needed for correlation (e.g., messageId, threadId, deep link, classification outputs).
  - Apply TTL/pruning to ODS tables; retain only non-PII metadata required for audit and idempotency.

### Duplicate Prevention & Thread Awareness

- Thread-aware context is grouped by `ThreadId` before AI analysis to ensure consolidated outputs per conversation.
- Dedupe key is Gmail `messageId` (and optionally `threadId`) to prevent re-processing.
- If Gmail labeling fails, use Gmail message ID as an idempotency key in downstream artifacts (e.g., Telegram correlation record; and later Google Tasks notes when Epic 3 is implemented).

### Error Handling Strategy

- Use **"Continue on Fail"** setting on LLM nodes for retryable errors (timeouts, rate limits)
- Inline Code node after LLM nodes validates response and returns partial results with error status
- Cap LLM input size; skip oversized items with partial status and reason
- When runs fail, log only non-PII metadata via Error Trigger workflow/strategy (no raw content)

### Task Card Delivery (current state vs deferred sync)

- In current workflows folder, **Task Cards are delivered to Telegram for review and HITL navigation** (not yet to Google Tasks).
- **Telegram Intelligence Delivery**: Summarized reports of daily activity (Tasks, FYI, Noise counts) and critical action items are sent to a designated Telegram chat by `Waypoint-Collect_TG.json`.
- **Core pipeline sequence (implemented)**:
  Email Ingestion (Buffered) → Telegram HITL Approve/Decline → Telegram Intelligence Report / Navigation Hub

### Google Tasks Field Mapping (Deferred)

- The following mapping remains the target design for a future “Tasks Sync” workflow, but is **not implemented** in the currently uploaded `workflows/` folder:
  - `title` → Google Task `title`
  - `dueDate` (ISO 8601 date: YYYY-MM-DD) → convert to RFC 3339 and map to Google Task `due`
  - `stakeholders` + `actionItems` + `sourceLink` → Google Task `notes` (formatted text)
  - If conversion fails, omit `due` and record statusReason

---

## Clarifications

### Session 2026-01-21

- Q: How should stateless processing be enforced when runs fail? → A: Disable execution data saving for all runs and log only non-PII metadata via Error Trigger workflow
- Q: How should duplicates be prevented if Gmail labeling fails? → A: Use Gmail message ID as an idempotency key in downstream artifacts (and in Google Tasks notes when sync is implemented); retry labeling separately
- Q: How should oversized emails be handled for n8n AI Agent limits? → A: Cap input size; skip oversized items with partial status and reason
- Q: How should date conversion be handled before Google Tasks sync? → A: Add an explicit transformation step to convert ISO 8601 (YYYY-MM-DD) to RFC 3339; if conversion fails, omit due date and record statusReason

### Session 2026-01-19

- Q: How will Gmail OAuth flow be handled in n8n? → A: Use n8n's built-in Gmail node with native OAuth2 credentials
- Q: How should Noise Filtering be implemented in n8n? → A: Use n8n's AI Agent node with lightweight LLM call
- Q: What is the n8n deployment mode for stateless processing? → A: Self-hosted Docker with execution data pruning
- Q: How should LLM errors be handled in n8n? → A: Combine "Continue on Fail" for retryable errors with Error Trigger for permanent failures
- Q: Where are Task Cards delivered? → A: Delivered to Telegram for HITL review in current implementation; Google Tasks delivery is deferred until a sync workflow is added.
- Q: When should Google Tasks sync occur? → A: Deferred; when implemented, it MUST occur only after explicit user approval via Telegram (HITL gating).
- Q: How to handle OAuth for Google Tasks? → A: Extend Gmail OAuth credential to include Tasks scope
- Q: How to prevent duplicate Google Tasks? → A: Add Gmail label "WAYPOINT-Processed" after successful sync (deferred)
- Q: How to map Task Card fields to Google Tasks? → A: title→title, dueDate (converted to RFC3339)→due, stakeholders+actionItems+sourceLink→notes (deferred)
- Q: How to handle Google Tasks sync failures? → A: Set status to "partial" with statusReason describing the error (deferred)

---

## Requirements *(mandatory)*

### Functional Requirements

#### Epic 1: Signal Acquisition

- **FR-001**: System MUST support Gmail OAuth 2.0 authorization using n8n's built-in Gmail credentials (no custom OAuth/PKCE implementation required).
- **FR-002**: System MUST fetch only unread emails from the user's inbox (no drafts, sent, or spam).
- **FR-003**: System MUST support both scheduled polling (configurable interval) and manual trigger for email fetching.
- **FR-004**: Token refresh MUST be handled by n8n's credential system (no custom refresh logic).
- **FR-005**: System MUST store fetched signals in a temporary `Mail_Table` buffer (Asia/Taipei timezone for timestamps).
- **FR-006**: System MUST perform thread-level grouping using `ThreadId` before passing data to the Intelligence stage.
- **FR-007**: System MUST detect and skip duplicate Message IDs already present in the `Mail_Table`.

#### Epic 2: Intelligence & Navigation

- **FR-011**: System MUST classify email intent into: "Actionable", "FYI", "Forward", or "Noise".
- **FR-012**: System MUST use an AI Agent to merge thread history (Old summaries + New snippets) for holistic context.
- **FR-013**: System MUST extract structured waypoint fields into a Task Card: title, dueDate (ISO 8601 date: YYYY-MM-DD or null), stakeholders[], actionItems[], sourceLink.
- **FR-014**: System MUST upsert derived results (no raw email content) into an `LLM_Table` and/or `Task_Table` for downstream HITL navigation and dedupe.

#### Epic 3: Synchronization (Deferred in current workflows folder)

- **FR-022 (Deferred)**: System MUST use an "AI Sync Expert" to reconcile `Task_Table` entries with actual Google Tasks.
- **FR-023 (Deferred)**: System MUST support three sync actions: **Create**, **Update**, and **Skip**.
- **FR-024 (Deferred)**: System MUST map Task Card fields to Google Tasks: title→title, dueDate→due (RFC 3339), stakeholders+actionItems+sourceLink→notes.
- **FR-025 (Deferred)**: System MUST update the `create_task_status` after successful synchronization.

---

## Task Card JSON Schema

The Task Card is the primary output artifact. It adheres to the **Stateless Processing** principle—containing only derived data, never raw email content.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "TaskCard",
  "description": "Structured task output from WAYPOINT email processing",
  "type": "object",
  "required": ["id", "title", "sourceLink", "intent", "createdAt", "status"],
  "properties": {
    "id": { "type": "string", "description": "Unique identifier for the task card (UUID v4)" },
    "title": { "type": "string", "description": "One-sentence summary of the task (max 120 characters)", "maxLength": 120 },
    "sourceLink": { "type": "string", "format": "uri", "description": "Gmail deep link to original email (https://mail.google.com/...)" },
    "intent": {
      "type": "string",
      "enum": ["Actionable", "FYI", "Forward"],
      "description": "Classified intent of the source email (Noise is excluded from TaskCard generation)"
    },
    "dueDate": {
      "type": ["string", "null"],
      "format": "date",
      "description": "Extracted due date in ISO 8601 format (YYYY-MM-DD), null if not found"
    }
    // ...existing code...
  },
  "additionalProperties": false
}
```

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

**Primary end-to-end success indicators (current implementation focus)**: SC-002, SC-007, SC-008, SC-014.

- **SC-001**: Users can complete Gmail OAuth authorization in under 60 seconds.
- **SC-002**: System processes 100 emails and generates task cards in under 5 minutes.
- **SC-003**: Noise filtering achieves at least 90% accuracy in identifying promotional/newsletter emails.
- **SC-004**: Intent classification correctly categorizes emails as Actionable/FYI/Forward at least 85% of the time (validated against human-labeled test set).
- **SC-005**: Waypoint extraction successfully identifies due dates when present in at least 80% of test cases.
- **SC-006**: System handles LLM provider unavailability gracefully—returns error status within 10 seconds, no system crash.
- **SC-007**: Zero raw email content or PII is persisted after processing completes (auditable via log review + execution data disabled/pruned).
- **SC-008**: All generated Task Cards conform to the defined JSON Schema (100% schema validation pass rate).
- **SC-009**: Deep links in Task Cards correctly navigate to the original Gmail message in 100% of cases.
- **SC-010**: System operates correctly on self-hosted n8n Docker deployment with execution data pruning configured.
- **SC-014**: Telegram notifications are delivered within 5 minutes of the scheduled trigger time with 100% template compliance.

**Deferred (requires Google Tasks sync workflow not present in current `workflows/` folder):**
- **SC-011 (Deferred)**: Google Tasks sync successfully creates tasks for 95%+ of Task Cards with status "success".
- **SC-012 (Deferred)**: Synced tasks contain correct title/due/notes mapping.
- **SC-013 (Deferred)**: Emails successfully synced receive "WAYPOINT-Processed" label preventing duplicate task creation.

---

## Dependencies

- **Gmail API**: For OAuth authorization, email fetching, and labeling (via n8n Gmail node).
- **Telegram Bot API**: For HITL navigation and notifications.
- **LLM Provider API**: For intent classification and waypoint extraction (via n8n AI Agent node).
- **Docker**: For self-hosted n8n deployment.
- **n8n**: Workflow automation platform (self-hosted via Docker).

- **Google Tasks API (Deferred)**: For task creation/sync (requires an additional workflow not present in current repo snapshot).