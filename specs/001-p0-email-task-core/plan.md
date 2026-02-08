# Implementation Plan: P0 Email-to-Task Core Pipeline

**Branch**: `001-p0-email-task-core` | **Date**: 2026-01-21 | **Spec**: [specs/001-p0-email-task-core/spec.md](specs/001-p0-email-task-core/spec.md)
**Input**: Feature specification from `/specs/001-p0-email-task-core/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

Deliver a modular n8n architecture that ingests Gmail, buffers data in internal tables, and uses specialized AI agents to analyze email threads and synchronize tasks with Human-in-the-Loop (HITL) approval. The pipeline consists of five core workflows: **Retriever** (Ingestion), **Classifier** (Thread-level Intelligence), **Intelligence Delivery** (Telegram Hub), **Task Sync** (AI-driven Deduplication), and **HITL Navigation** (Approve/Decline Flow). The system uses a buffered architecture (`Mail_Table`, `LLM_Table`, `Task_Table`) to enable robust thread merging and state management while maintaining privacy-conscious processing.

## Technical Context

**Language/Version**: n8n workflow JSON (modular multi-workflow architecture)  
**Primary Dependencies**: n8n (self-hosted Docker), Gmail API, Google Tasks API, Telegram Bot API, OpenAI GPT-5-nano (LLM)  
**Storage**: n8n Internal Data Tables (`Mail_Table`, `LLM_Table`, `Task_Table`) used as a temporary Operational Data Store (ODS).  
**Testing**: Modular workflow execution tests + AI Agent logic validation.  
**Target Platform**: Self-hosted n8n on Linux Docker  
**Project Type**: modular (separate workflows for ingestion, analysis, sync, notification, and human-in-the-loop approval)  
**Performance Goals**: Process 100 emails and generate/sync tasks in <5 minutes (SC-002)  
**Constraints**: Thread-level context merging, AI-driven idempotency, Asia/Taipei timezone handling, HITL approval.  
**Scale/Scope**: P0 core intelligence pipeline supporting multi-email thread deduplication and user-driven task approval.

## Constitution Check

- **Principle I (Code Quality)**: PASS — Workflows are modularized by responsibility (Ingest, Classify, Sync, Notify, Approve).
- **Principle II (Testing Standards)**: PASS — Independent testing possible for each module via the ODS tables.
- **Principle III (User Experience)**: PASS — Proactive Telegram summaries with interactive buttons provide high-density intelligence delivery and control.
- **Principle IV (MVP & Anti-Over-Design)**: PASS — No external frontend; leverage native Google Tasks and Telegram UI.
- **Principle V (Stateless Processing)**: PARTIAL — Uses internal tables for buffering thread context; execution data pruning remains active.
- **Principle VI (Integration Flexibility)**: PASS — Modular design allows replacing individual AI Agents or providers.
- **Principle VII (Error Handling)**: PASS — Status tracking in DB tables and dedicated Error Trigger workflow allow for robust retry and auditing.

## Project Structure

### Documentation

```text
specs/001-p0-email-task-core/
├── plan.md              # This file
├── spec.md              # Feature specification
└── tasks.md             # Task list and progress tracking
```

### Source Code

```text
workflows/
├── Waypoint-mail-retriever.json              # Ingestion module
├── Waypoint-classifier.json                  # Intelligence module (Thread analysis) - Integrated in Collect_TG
├── Waypoint-Collect_TG.json                  # Integrated Intelligence & Notification module
├── Waypoint-task.json                        # Synchronization module (Sync Expert)
├── Waypoint-Telegram-Approve Or Decline Flow.json # HITL Navigation module
└── waypoint-error-trigger.json               # Error Handling & Compliance module
```

**Structure Decision**: Documentation-driven plan; workflow JSON will be stored under workflows/ when implemented.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| [e.g., 4th project] | [current need] | [why 3 projects insufficient] |
| [e.g., Repository pattern] | [specific problem] | [why direct DB access insufficient] |
