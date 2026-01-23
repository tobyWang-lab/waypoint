---

description: "Task list for P0 Email-to-Task Core Pipeline"
---

# Tasks: P0 Email-to-Task Core Pipeline

**Input**: Design documents from `/specs/001-p0-email-task-core/`
**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Tests**: Tests are included because the feature requires workflow execution validation and compliance checks.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **Single project**: `workflows/`, `specs/` at repository root

---

## Phase 1: Environment & Credentials (Setup)

**Purpose**: n8n Docker setup requirements and credential prerequisites

- [X] T001 Create base workflow file at workflows/waypoint-email-to-tasks.json with workflow metadata and empty node graph. DoD: file exists with valid n8n JSON structure; Verify: open and validate JSON structure in workflows/waypoint-email-to-tasks.json.
- [X] T002 [P] Document required n8n environment settings (including N8N_EXECUTION_PROCESS_DATA_PRUNING and execution data saving disabled) in specs/001-p0-email-task-core/quickstart.md. DoD: env var names and required values are listed; Verify: review updated section in specs/001-p0-email-task-core/quickstart.md.
- [X] T003 Add credential placeholders for Gmail/Google Tasks OAuth and LLM provider in workflows/waypoint-email-to-tasks.json. DoD: workflow nodes reference credential names/scopes; Verify: inspect credential blocks in workflows/waypoint-email-to-tasks.json.

---

## Phase 2: Ingestion & Pre-processing — User Story 1 (Priority: P1) 🎯 MVP

**Goal**: Secure Gmail OAuth authorization and unread inbox ingestion with pre-processing safeguards.

**Independent Test**: Authorize Gmail and verify unread inbox messages are ingested without persisting raw content.

### Implementation for User Story 1

- [X] T004 [US1] Configure Gmail trigger/polling node to fetch unread inbox emails and exclude label "WAYPOINT-Processed" in workflows/waypoint-email-to-tasks.json. DoD: Gmail node filters unread inbox and label exclusion; Verify: node query/filters in workflows/waypoint-email-to-tasks.json.
- [X] T005 [US1] Add Gmail message detail retrieval (body + attachment metadata) in workflows/waypoint-email-to-tasks.json. DoD: message payload includes body and attachment metadata; Verify: node output mapping in workflows/waypoint-email-to-tasks.json.
- [X] T006 [US1] Add Code node to cap email body length and attachment bytes, setting statusReason on oversize in workflows/waypoint-email-to-tasks.json. DoD: size caps applied and oversize flagged as partial; Verify: code logic present in workflows/waypoint-email-to-tasks.json.

**Checkpoint**: User Story 1 is independently testable and enforces stateless ingestion.

---

## Phase 3: AI Intelligence — User Stories 2–4 (Priority: P1)

### User Story 2 - Noise Filtering (Priority: P1)

**Goal**: Filter noise and pass only signal to downstream processing.

**Independent Test**: Ingest known noise emails and verify they are classified as Noise and skipped.

- [X] T007 [US2] Add AI Agent node prompt for noise classification and category output in workflows/waypoint-email-to-tasks.json. DoD: prompt yields Noise/Signal + category; Verify: prompt text and output schema in workflows/waypoint-email-to-tasks.json.
- [X] T008 [US2] Add routing logic to short-circuit Noise items with status "skipped" in workflows/waypoint-email-to-tasks.json. DoD: Noise items do not proceed to intent/extraction; Verify: IF/Switch logic in workflows/waypoint-email-to-tasks.json.

### User Story 3 - Intent Classification (Priority: P1)

**Goal**: Classify signal emails as Actionable/FYI/Forward.

**Independent Test**: Submit sample emails and verify intent classification accuracy.

- [X] T009 [US3] Add AI Agent node (or prompt branch) for intent classification with confidence in workflows/waypoint-email-to-tasks.json. DoD: intent output includes Actionable/FYI/Forward; Verify: node configuration in workflows/waypoint-email-to-tasks.json.

### User Story 4 - Waypoint Extraction (Priority: P1)

**Goal**: Extract dueDate, stakeholders, and actionItems into a Task Card.

**Independent Test**: Provide emails with known fields and verify extracted Task Card fields.

- [X] T010 [US4] Add AI Agent extraction prompt to produce Task Card fields in workflows/waypoint-email-to-tasks.json. DoD: extraction output includes dueDate/stakeholders/actionItems; Verify: prompt and output mapping in workflows/waypoint-email-to-tasks.json.
- [X] T011 [US4] Validate Task Card output against contracts/task-card.schema.json in workflows/waypoint-email-to-tasks.json. DoD: invalid outputs set partial status with statusReason; Verify: schema validation logic and contract path in workflows/waypoint-email-to-tasks.json.

**Checkpoint**: User Stories 2–4 are independently testable and yield valid Task Cards.

---

## Phase 4: Sync & Deduplication — User Story 5 (Priority: P0)

**Goal**: Sync actionable Task Cards to Google Tasks with idempotency and labeling.

**Independent Test**: Create a Task Card and confirm Google Task creation, RFC 3339 due date, and Gmail labeling.

- [ ] T012 [US5] Add transform step to convert ISO 8601 dueDate to RFC 3339 (omit on failure) in workflows/waypoint-email-to-tasks.json. DoD: conversion applied or omitted with statusReason; Verify: transform logic in workflows/waypoint-email-to-tasks.json.
- [ ] T013 [US5] Add Google Tasks lookup/list step to check idempotency key in notes before create in workflows/waypoint-email-to-tasks.json. DoD: existing key skips create; Verify: list+filter logic in workflows/waypoint-email-to-tasks.json.
- [ ] T014 [US5] Add Google Tasks create node mapping to contracts/google-tasks-sync.request.json in workflows/waypoint-email-to-tasks.json. DoD: title/notes/due mapping matches contract; Verify: node fields and mapping in workflows/waypoint-email-to-tasks.json.
- [ ] T015 [US5] Add Gmail label node to apply "WAYPOINT-Processed" after successful sync in workflows/waypoint-email-to-tasks.json. DoD: label applied only on sync success; Verify: node placement/conditions in workflows/waypoint-email-to-tasks.json.
- [ ] T016 [US5] Create Error Trigger workflow in workflows/waypoint-error-trigger.json emitting non-PII events per contracts/error-event.schema.json. DoD: error workflow references schema fields and excludes PII; Verify: review workflows/waypoint-error-trigger.json.

**Checkpoint**: User Story 5 is independently testable and delivers tasks to Google Tasks with deduplication.

---

## Phase 5: Testing & Compliance

**Purpose**: Validate workflow behavior and stateless privacy compliance.

- [ ] T017 Create workflow execution test checklist in specs/001-p0-email-task-core/testing.md covering US1–US5 acceptance scenarios. DoD: checklist maps to each story’s independent test; Verify: review specs/001-p0-email-task-core/testing.md.
- [ ] T018 [P] Create stateless privacy audit checklist in specs/001-p0-email-task-core/compliance.md (no execution data saved, no PII in logs). DoD: audit steps and pass criteria documented; Verify: review specs/001-p0-email-task-core/compliance.md.
- [ ] T019 Update specs/001-p0-email-task-core/quickstart.md to reference testing and compliance checklists. DoD: quickstart links to testing/compliance docs; Verify: review specs/001-p0-email-task-core/quickstart.md.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Environment & Credentials)**: No dependencies - can start immediately
- **Phase 2 (Ingestion & Pre-processing)**: Depends on Phase 1
- **Phase 3 (AI Intelligence)**: Depends on Phase 2
- **Phase 4 (Sync & Deduplication)**: Depends on Phase 3
- **Phase 5 (Testing & Compliance)**: Depends on Phases 1–4

### User Story Dependencies

- **US1 (P1)**: Depends on Phase 1
- **US2 (P1)**: Depends on US1 output (signal emails)
- **US3 (P1)**: Depends on US2 output (signal routing)
- **US4 (P1)**: Depends on US3 output (intent)
- **US5 (P0)**: Depends on US4 output (Task Cards) and shared transforms

### Within Each User Story

- Implement ingestion before AI steps
- AI outputs before validation
- Validation before sync
- Sync before labeling

### Parallel Opportunities

- T002 and T018 can run in parallel (different files)
- US2 and US3 prompt edits can be prepared in parallel before integration
- Documentation tasks in Phase 5 can run in parallel

---

## Parallel Example: User Story 2

```bash
# Parallel prompt and routing tasks:
Task: "Add AI Agent node prompt for noise classification" (T007)
Task: "Add routing logic to short-circuit Noise items" (T008)
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1
2. Complete Phase 2 (US1)
3. **STOP and VALIDATE**: Test US1 independently using Gmail ingestion

### Incremental Delivery

1. Phase 1 → Phase 2 (US1) → Validate
2. Phase 3 (US2–US4) → Validate
3. Phase 4 (US5) → Validate
4. Phase 5 (Testing & Compliance)

### Parallel Team Strategy

- One owner handles workflow JSON updates
- Another owner prepares testing/compliance docs in parallel

---

## Notes

- [P] tasks = different files, no dependencies
- Each user story is independently testable using the listed acceptance scenarios
- Avoid introducing stateful storage or dashboard features
