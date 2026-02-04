# Feature Specification: P0 Email-to-Task Core Pipeline

**Feature Branch**: `001-p0-email-task-core`  
**Created**: 2026-01-18  
**Status**: Draft  
**Input**: P0 Email-to-Task Core Pipeline covering Signal Acquisition (Epic 1) and Intelligence Navigation (Epic 2)

---

## Overview

This specification defines the P0 (Critical) features required to build the core "Email-to-Task" pipeline for WAYPOINT. It covers:

- **Epic 1 - Signal Acquisition**: OAuth Gmail integration and email ingestion
- **Epic 2 - Intelligence & Navigation**: Thread-aware intent classification, noise filtering, and waypoint extraction using LLM
- **Epic 3 - Synchronization**: AI-driven Google Tasks state synchronization
- **Epic 4 - Notification**: Telegram intelligence delivery

The **core pipeline sequence** is: Email Ingestion (Buffered) → Thread-based AI Analysis (Classifier) → AI-driven Task Synchronization (Sync Expert) → Telegram Notification Summary.

All features adhere to the Constitution's **Stateless Processing** principle—no email content persists beyond processing.

## Target Environment: n8n

This specification defines the Email-to-Task pipeline as an n8n workflow. All components MUST:
- Be deployable as n8n workflow nodes or steps
- Use **n8n native nodes** for all interactions (Gmail, Google Tasks, LLM via AI Agent/HTTP)
- Use n8n's HTTP request capabilities for external service integration
- Leverage n8n's credential management for OAuth and API authentication
- Be exported and versioned in n8n's JSON workflow format
- Be testable within n8n's workflow execution environment

### Gmail Integration

- Use n8n's **built-in Gmail node** with native OAuth2 credentials
- Token refresh is handled automatically by n8n's credential system
- No custom OAuth2 flow implementation required
- OAuth credential includes **Google Tasks scope** (`https://www.googleapis.com/auth/tasks`) for unified authorization

### Noise Filtering Implementation

- Use n8n's **AI Agent node** with a lightweight LLM call for noise classification
- LLM determines Noise vs Signal and categorizes noise type (Advertisement, Newsletter, Notification)
- Single unified AI approach for both noise filtering and intent classification

### Deployment & Stateless Processing

- Deploy as **self-hosted n8n via Docker**
- Environment-agnostic and container-ready per **Constitution v2.0.0**
- **Buffered Data Architecture**: Uses internal n8n Data Tables (`Mail_Table`, `LLM_Table`, `Task_Table`) as a temporary Operational Data Store (ODS) to enable thread-level context merging and robust synchronization.
- **Stateless Intent**: While ODS tables are used for processing batches, the long-term goal remains to prune execution data and minimize persistent PII.
- Execution history containing email data is auto-deleted after workflow completion (pruning enabled)
- No long-term persistent storage of raw email content beyond the current processing cycle.

### Duplicate Prevention & Thread Awareness

- **AI-Driven Synchronization**: Uses a "Task Synchronization Expert" AI Agent to reconcile newly extracted waypoints with existing records in the `Task_Table` and Google Tasks.
- **Thread-Aware Context**: Multiple emails within the same `ThreadId` are merged before AI analysis to ensure a single, consolidated task represents the entire conversation history.
- **Idempotency**: The system handles "Update" vs "Insert" logic by comparing `ThreadId` and existing Task IDs.
- Maintains stateless processing while preventing duplicate task creation per thread.

### Error Handling Strategy

- Use **"Continue on Fail"** setting on LLM nodes for retryable errors (timeouts, rate limits)
- Use **Error Trigger workflow** to catch and log permanent failures (auth errors, malformed responses)
- Inline Code node after LLM nodes validates response and returns partial results with error status
- Cap LLM input size; skip oversized items with partial status and reason

### Task Card Delivery

- Task Cards are delivered **directly to Google Tasks** with no intermediate Dashboard
- n8n workflow remains stateless and headless; no external frontend persistence is required
- **Telegram Intelligence Delivery**: Summarized reports of daily activity (Tasks, FYI, Noise counts) and critical action items are sent to a designated Telegram chat.
- **Core pipeline sequence**: Email Ingestion (Buffered) → Thread-based AI Analysis (Classifier) → AI-driven Task Synchronization (Sync Expert) → Telegram Notification Summary

### Google Tasks Field Mapping

- `title` → Google Task `title`
- `dueDate` → Google Task `due` (RFC 3339 format)
- `stakeholders` + `actionItems` + `sourceLink` → Google Task `notes` (formatted text)
- Convert ISO 8601 dates to RFC 3339 before sync; if conversion fails, omit `due` and record statusReason

---

## Clarifications

### Session 2026-01-21

- Q: How should stateless processing be enforced when runs fail? → A: Disable execution data saving for all runs and log only non-PII metadata via Error Trigger workflow
- Q: How should duplicates be prevented if Gmail labeling fails? → A: Use Gmail message ID as an idempotency key in Google Tasks notes and skip creation if the key already exists; retry labeling separately
- Q: How should oversized emails be handled for n8n AI Agent limits? → A: Cap input size; skip oversized items with partial status and reason
- Q: How should date conversion be handled before Google Tasks sync? → A: Add an explicit transformation step to convert ISO 8601 to RFC 3339; if conversion fails, omit due date and record statusReason

### Session 2026-01-19

- Q: How will Gmail OAuth flow be handled in n8n? → A: Use n8n's built-in Gmail node with native OAuth2 credentials
- Q: How should Noise Filtering be implemented in n8n? → A: Use n8n's AI Agent node with lightweight LLM call
- Q: What is the n8n deployment mode for stateless processing? → A: Self-hosted Docker with execution data pruning
- Q: How should LLM errors be handled in n8n? → A: Combine "Continue on Fail" for retryable errors with Error Trigger for permanent failures
- Q: Where are Task Cards delivered? → A: Directly to Google Tasks (no Dashboard or Webhook delivery)
- Q: When should Google Tasks sync occur? → A: Immediately after successful LLM extraction
- Q: How to handle OAuth for Google Tasks? → A: Extend Gmail OAuth credential to include Tasks scope
- Q: How to prevent duplicate Google Tasks? → A: Add Gmail label "WAYPOINT-Processed" after successful sync
- Q: How to map Task Card fields to Google Tasks? → A: title→title, dueDate→due, stakeholders+actionItems+sourceLink→notes
- Q: How to handle Google Tasks sync failures? → A: Set status to "partial" with statusReason describing the error

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Gmail OAuth Authorization & Email Ingestion (Priority: P1)

As a user, I want the system to securely connect to my Gmail via OAuth and fetch my unread emails, so that I can focus on the latest information without manual copy-paste operations.

**Why this priority**: This is the foundational data pipeline. Without email ingestion, no subsequent analysis or task generation can occur. It is the entry point for all downstream features.

**Independent Test**: Can be fully tested by authorizing a Gmail account and verifying that unread emails are retrieved correctly. Delivers value by enabling automated email collection.

**Acceptance Scenarios**:

1. **Given** a user has not authorized the system, **When** they initiate the OAuth flow, **Then** they are redirected to Google's consent screen and can grant access.
2. **Given** a user has authorized the system, **When** the system fetches emails, **Then** only unread emails from the inbox are retrieved.
3. **Given** a user has authorized the system, **When** the OAuth token expires, **Then** the system automatically refreshes the token without user intervention.
4. **Given** a user has authorized the system, **When** email fetch completes, **Then** no raw email content is persisted to disk or database (stateless processing).

---

### User Story 2 - Noise Filtering (Priority: P1)

As an efficiency-focused user, I want the system to automatically filter out advertisements, system notifications, and newsletters before deep analysis, so that AI resources are only spent on valuable emails and my task list is not cluttered with junk.

**Why this priority**: Noise filtering is essential to reduce LLM cost and ensure task quality. Without filtering, the system would generate useless tasks from promotional emails.

**Independent Test**: Can be fully tested by ingesting a batch of emails containing known noise (newsletters, promotions) and verifying they are marked as "Noise" and excluded from further processing.

**Acceptance Scenarios**:

1. **Given** an email from a promotional sender, **When** noise detection runs, **Then** the email is classified as "Noise" with category "Advertisement".
2. **Given** an email with subject matching newsletter patterns, **When** noise detection runs, **Then** the email is classified as "Noise" with category "Newsletter".
3. **Given** an automated system notification email, **When** noise detection runs, **Then** the email is classified as "Noise" with category "Notification".
4. **Given** a legitimate business email, **When** noise detection runs, **Then** the email passes through and is marked as "Signal".
5. **Given** noise filtering has completed, **When** results are returned, **Then** no email content is stored—only the classification result and email ID are retained temporarily.

---

### User Story 3 - Intent Classification (Priority: P1)

As a user, I want the AI to classify each email as "Actionable", "FYI", or "Forward", so that I can focus only on emails requiring action and convert them into tasks.

**Why this priority**: Intent classification is the core intelligence that determines whether an email becomes a task. This directly enables the Email-to-Task conversion.

**Independent Test**: Can be fully tested by submitting sample emails with clear intents and verifying correct classification. Delivers value by automatically categorizing emails for task routing.

**Acceptance Scenarios**:

1. **Given** an email containing a direct request with a deadline, **When** intent classification runs, **Then** the email is classified as "Actionable".
2. **Given** an email that is informational with no required action, **When** intent classification runs, **Then** the email is classified as "FYI".
3. **Given** an email asking to forward to another person, **When** intent classification runs, **Then** the email is classified as "Forward".
4. **Given** the LLM service is unavailable, **When** intent classification is attempted, **Then** the system returns a null result with an error status (graceful degradation).
5. **Given** classification completes, **When** results are returned, **Then** no email content is persisted—only the classification output is retained temporarily.

---

### User Story 4 - Waypoint Extraction (Priority: P1)

As a project team member, I want the system to automatically extract "due date", "key stakeholders", and "specific action items" from email content, so that tasks are concrete and actionable rather than vague email subjects.

**Why this priority**: This is the core value of WAYPOINT—transforming unstructured emails into structured, actionable data. Without extraction, task cards would be empty shells.

**Independent Test**: Can be fully tested by submitting emails with known deadlines, names, and action items, then verifying the extracted fields match expectations.

**Acceptance Scenarios**:

1. **Given** an email containing "Please complete by January 25th", **When** waypoint extraction runs, **Then** the due date is extracted as "2026-01-25".
2. **Given** an email mentioning "@John Smith as lead", **When** waypoint extraction runs, **Then** "John Smith" is identified as a key stakeholder.
3. **Given** an email stating "Please review the Q4 report and send feedback", **When** waypoint extraction runs, **Then** the action item is extracted as "Review Q4 report and send feedback".
4. **Given** an email with no clear deadline, **When** waypoint extraction runs, **Then** the due date field is null (not fabricated).
5. **Given** the LLM response is malformed, **When** parsing fails, **Then** the system logs the error and returns a partial result with error indication.
6. **Given** extraction completes, **When** results are returned, **Then** no raw email content is stored—only the structured Task Card output is retained temporarily.

---

### User Story 5 - Google Tasks Synchronization (Priority: P0)

As a user, I want every approved actionable task to be automatically synced to my Google Tasks, so that my to-do list is unified in a tool I already use daily.

**Why this priority**: Google Tasks sync completes the Email-to-Task pipeline by delivering tasks to the user's native task management system. Without this, users must manually recreate tasks.

**Independent Test**: Can be fully tested by generating a Task Card and verifying it appears in Google Tasks with correct title, due date, and notes.

**Acceptance Scenarios**:

1. **Given** a Task Card with status "success", **When** Google Tasks sync runs, **Then** a new task is created in the user's default Google Tasks list.
2. **Given** a Task Card with title and dueDate, **When** synced to Google Tasks, **Then** the task title matches and due date is in RFC 3339 format.
3. **Given** a Task Card with stakeholders, actionItems, and sourceLink, **When** synced, **Then** these are combined into the Google Task notes field.
4. **Given** an email that was successfully synced, **When** sync completes, **Then** the email receives the "WAYPOINT-Processed" Gmail label.
5. **Given** Google Tasks API returns an error, **When** sync fails, **Then** Task Card status is set to "partial" with statusReason explaining the failure.
6. **Given** an email already has "WAYPOINT-Processed" label, **When** workflow runs, **Then** the email is skipped to prevent duplicate tasks.

---

### Edge Cases

- What happens when Gmail API rate limit is exceeded? → System backs off exponentially and retries, returning partial results with status indication.
- What happens when an email has no body text? → Noise filter processes based on headers only; if passed to LLM, extraction returns empty fields with "Insufficient content" status.
- What happens when email is in a non-English language? → LLM attempts classification; if confidence is low, task is flagged for manual review.
- What happens when an email contains multiple action items? → Extraction returns an array of action items, each as a separate waypoint.
- What happens when OAuth token is revoked by user? → System detects 401 error, marks user as "deauthorized", and prompts re-authorization on next access.
- What happens when email body exceeds LLM limits? → Skip oversized content and return partial status with explicit size-limit reason.
- What happens when Google Tasks API rate limit is exceeded? → Set Task Card status to "partial" with statusReason; workflow can retry per n8n error handling.
- What happens when dueDate format is invalid for Google Tasks? → Convert to RFC 3339 format; if conversion fails, sync without due date and note in statusReason.
- What happens when Google Tasks sync succeeds but Gmail labeling fails? → Log warning; retry Gmail labeling and prevent duplicates by checking a Gmail message ID idempotency key stored in Google Tasks notes.

---

## Requirements *(mandatory)*

### Functional Requirements

#### Epic 1: Signal Acquisition

- **FR-001**: System MUST support OAuth 2.0 authorization flow with Gmail API using PKCE for security.
- **FR-002**: System MUST fetch only unread emails from the user's inbox (no drafts, sent, or spam).
- **FR-003**: System MUST support both scheduled polling (configurable interval) and manual trigger for email fetching.
- **FR-004**: System MUST implement token refresh logic to maintain persistent access without user re-authorization.
- **FR-005**: System MUST store fetched signals in a temporary `Mail_Table` buffer (Asia/Taipei timezone).
- **FR-006**: System MUST perform thread-level grouping using `ThreadId` before passing data to the Intelligence stage.
- **FR-007**: System MUST detect and skip duplicate Message IDs already present in the `Mail_Table`.

#### Epic 2: Intelligence & Navigation

- **FR-011**: System MUST classify email intent into: "Actionable", "FYI", "Forward", or "Noise".
- **FR-012**: System MUST use an AI Agent to merge thread history (Old summaries + New snippets) for holistic context.
- **FR-013**: System MUST extract structured waypoints: task_subject, task_content, due_date (RFC 3339).
- **FR-014**: System MUST upsert results into an `LLM_Table` and `Task_Table` for downstream synchronization.

#### Epic 3: Synchronization

- **FR-022**: System MUST use an "AI Sync Expert" to reconcile `Task_Table` entries with actual Google Tasks.
- **FR-023**: System MUST support three sync actions: **Create** (New thread), **Update** (New info on existing thread), and **Skip** (Duplicate).
- **FR-024**: System MUST map Task Card fields to Google Tasks: title→title, dueDate→due (RFC 3339), summary+sourceLink→notes.
- **FR-025**: System MUST update the `create_task_status` in the DB after successful synchronization.

### Key Entities

- **Email Signal**: Represents an email that passed noise filtering. Contains message ID, subject, sender, timestamp, and processing status.
- **Task Card**: The structured output of the pipeline. Contains title, due date, stakeholders, action items, source link, and metadata.
- **Noise Classification**: Result of noise filtering. Contains message ID, noise category, and confidence score.
- **Intent Classification**: Result of intent analysis. Contains message ID, intent type (Actionable/FYI/Forward), and confidence score.
- **Waypoint Extraction**: Result of LLM extraction. Contains due date, stakeholders, action items, and extraction confidence.

### Task Card JSON Schema

The Task Card is the primary output artifact. It adheres to the **Stateless Processing** principle—containing only derived data, never raw email content.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "TaskCard",
  "description": "Structured task output from WAYPOINT email processing",
  "type": "object",
  "required": ["id", "title", "sourceLink", "intent", "createdAt", "status"],
  "properties": {
    "id": {
      "type": "string",
      "description": "Unique identifier for the task card (UUID v4)"
    },
    "title": {
      "type": "string",
      "description": "One-sentence summary of the task (max 120 characters)",
      "maxLength": 120
    },
    "sourceLink": {
      "type": "string",
      "format": "uri",
      "description": "Gmail deep link to original email (https://mail.google.com/...)"
    },
    "intent": {
      "type": "string",
      "enum": ["Actionable", "FYI", "Forward"],
      "description": "Classified intent of the source email"
    },
    "dueDate": {
      "type": ["string", "null"],
      "format": "date",
      "description": "Extracted due date in ISO 8601 format (YYYY-MM-DD), null if not found"
    },
    "stakeholders": {
      "type": "array",
      "items": {
        "type": "string"
      },
      "description": "List of key people mentioned in the email"
    },
    "actionItems": {
      "type": "array",
      "items": {
        "type": "string"
      },
      "description": "List of specific actions to be taken"
    },
    "priority": {
      "type": ["string", "null"],
      "enum": ["High", "Medium", "Low", null],
      "description": "AI-suggested priority level (null if not estimated)"
    },
    "estimatedMinutes": {
      "type": ["integer", "null"],
      "minimum": 0,
      "description": "AI-estimated time to complete in minutes (null if not estimated)"
    },
    "category": {
      "type": "string",
      "enum": ["urgent", "followup", "blocked", "completed", "draft"],
      "description": "Task category per Constitution Principle III"
    },
    "createdAt": {
      "type": "string",
      "format": "date-time",
      "description": "Timestamp when the task card was generated (ISO 8601)"
    },
    "status": {
      "type": "string",
      "enum": ["success", "partial", "failure", "skipped"],
      "description": "Processing status per Constitution Principle III"
    },
    "statusReason": {
      "type": ["string", "null"],
      "description": "Human-readable explanation for non-success status"
    },
    "metadata": {
      "type": "object",
      "properties": {
        "emailMessageId": {
          "type": "string",
          "description": "Gmail message ID for reference (not the content)"
        },
        "processingDurationMs": {
          "type": "integer",
          "description": "Time taken to process this email in milliseconds"
        },
        "llmProvider": {
          "type": "string",
          "description": "LLM provider used for extraction (e.g., 'openai', 'anthropic')"
        },
        "extractionConfidence": {
          "type": "number",
          "minimum": 0,
          "maximum": 1,
          "description": "Overall confidence score for extraction accuracy"
        }
      }
    }
  },
  "additionalProperties": false
}
```

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

**Primary end-to-end success indicators**: SC-011 and SC-012.

- **SC-001**: Users can complete Gmail OAuth authorization in under 60 seconds.
- **SC-002**: System processes 100 emails and generates task cards in under 5 minutes.
- **SC-003**: Noise filtering achieves at least 90% accuracy in identifying promotional/newsletter emails.
- **SC-004**: Intent classification correctly categorizes emails as Actionable/FYI/Forward at least 85% of the time (validated against human-labeled test set).
- **SC-005**: Waypoint extraction successfully identifies due dates when present in at least 80% of test cases.
- **SC-006**: System handles LLM provider unavailability gracefully—returns error status within 10 seconds, no system crash.
- **SC-007**: Zero raw email content or PII is persisted after processing completes (auditable via log review).
- **SC-008**: All generated Task Cards conform to the defined JSON Schema (100% schema validation pass rate).
- **SC-009**: Deep links in Task Cards correctly navigate to the original Gmail message in 100% of cases.
- **SC-010**: System operates correctly on self-hosted n8n Docker deployment with execution data pruning configured.
- **SC-011**: Google Tasks sync successfully creates tasks in user's Google Tasks list for 95%+ of Task Cards with status "success".
- **SC-012**: Task Cards synced to Google Tasks contain correct title, due date, and notes with stakeholders/actionItems/sourceLink.
- **SC-013**: Emails successfully synced receive "WAYPOINT-Processed" label preventing duplicate task creation.
- **SC-014**: Telegram notifications are delivered within 5 minutes of the scheduled trigger time with 100% template compliance.

---

## Assumptions

- Users have a valid Gmail account with OAuth 2.0 access enabled.
- Users grant both Gmail and Google Tasks OAuth scopes during authorization.
- The LLM provider (OpenAI, Anthropic, or equivalent) is accessible via standard REST APIs.
- Email content is primarily in English (non-English support is best-effort with manual review flag).
- The system operates in a server environment with internet access to Gmail, Google Tasks, and LLM APIs.
- Self-hosted n8n via Docker is the target deployment environment per Constitution Technical Constraints.
- Execution data pruning is configured to enforce stateless processing per Constitution Principle V.
- Users have at least one Google Tasks list (default list used if not specified).

---

## Dependencies

- **Gmail API**: For OAuth authorization, email fetching, and labeling (via n8n Gmail node).
- **Google Tasks API**: For task creation and sync (via extended OAuth credentials).
- **Telegram Bot API**: For intelligence delivery and notifications.
- **LLM Provider API**: For intent classification and waypoint extraction (via n8n AI Agent node).
- **Docker**: For self-hosted n8n deployment.
- **n8n**: Workflow automation platform (self-hosted via Docker).
