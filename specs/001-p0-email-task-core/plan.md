# Implementation Plan: P0 Email-to-Task Core Pipeline

**Branch**: `001-p0-email-task-core` | **Date**: 2026-01-21 | **Spec**: [specs/001-p0-email-task-core/spec.md](specs/001-p0-email-task-core/spec.md)
**Input**: Feature specification from `/specs/001-p0-email-task-core/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

Deliver a headless n8n workflow that ingests Gmail, filters noise, extracts intent/waypoints with an LLM, syncs actionable tasks directly to Google Tasks, and labels processed emails. The workflow enforces stateless processing (no execution data saved), uses idempotency keys to prevent duplicates, caps LLM input size, and converts ISO 8601 dates to RFC 3339 before sync.

## Technical Context

<!--
  ACTION REQUIRED: Replace the content in this section with the technical details
  for the project. The structure here is presented in advisory capacity to guide
  the iteration process.
-->

**Language/Version**: n8n workflow JSON (no app runtime language)  
**Primary Dependencies**: n8n (self-hosted Docker), Gmail API, Google Tasks API, LLM provider API  
**Storage**: N/A (stateless; no persistent storage of raw email data)  
**Testing**: n8n workflow execution tests + schema validation of Task Cards  
**Target Platform**: Self-hosted n8n on Linux Docker  
**Project Type**: single (workflow configuration + documentation)  
**Performance Goals**: Process 100 emails and generate Task Cards in <5 minutes (SC-002)  
**Constraints**: Stateless processing, no execution data saved, LLM token/timeout limits, RFC 3339 conversion required  
**Scale/Scope**: P0 pipeline for single workflow run; batch size ~100 emails

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **Principle I (Code Quality)**: PASS — workflow nodes separated by responsibility (ingest, filter, extract, sync, label).
- **Principle II (Testing Standards)**: PASS — plan includes workflow execution tests and schema validation; accuracy tests required in implementation.
- **Principle III (User Experience)**: PASS — Task Card schema and status fields maintained per spec.
- **Principle IV (MVP & Anti-Over-Design)**: PASS — scope limited to email-to-task pipeline with no dashboards.
- **Principle V (Stateless Processing)**: PASS — execution data saving disabled; no raw email persistence.
- **Principle VI (Integration Flexibility)**: PASS — LLM provider configurable via standard APIs.
- **Principle VII (Error Handling)**: PASS — partial statuses, error triggers, and size caps defined.

**Post-Design Check (Phase 1)**: PASS — artifacts align with principles and do not introduce new storage or UI scope.

## Project Structure

### Documentation (this feature)

```text
specs/[###-feature]/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)

```text
workflows/
└── waypoint-email-to-tasks.json
```

**Structure Decision**: Documentation-driven plan; workflow JSON will be stored under workflows/ when implemented.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| [e.g., 4th project] | [current need] | [why 3 projects insufficient] |
| [e.g., Repository pattern] | [specific problem] | [why direct DB access insufficient] |
