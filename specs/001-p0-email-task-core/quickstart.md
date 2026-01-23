# Quickstart: P0 Email-to-Task Core Pipeline (n8n)

## Prerequisites
- Docker installed (Docker Engine 20.10+ recommended)
- n8n self-hosted instance (v1.0+ required)
- Google Cloud project with OAuth credentials configured for Gmail + Google Tasks scopes
- LLM provider API key compatible with n8n AI Agent node (OpenAI, Anthropic, or compatible REST)

## Environment Variables (RHEL/Linux Docker Host)

Configure the following environment variables for your n8n Docker container to enforce **Stateless Processing** per Constitution Principle V.

### Required n8n Environment Variables

| Variable | Value | Purpose |
|----------|-------|---------|
| `N8N_EXECUTIONS_DATA_SAVE_ON_ERROR` | `none` | Disable execution data saving on error |
| `N8N_EXECUTIONS_DATA_SAVE_ON_SUCCESS` | `none` | Disable execution data saving on success |
| `N8N_EXECUTIONS_DATA_SAVE_ON_PROGRESS` | `false` | Disable saving intermediate execution data |
| `N8N_EXECUTIONS_DATA_SAVE_MANUAL_EXECUTIONS` | `false` | Disable saving manual test executions |
| `N8N_EXECUTIONS_PROCESS_DATA_PRUNING` | `true` | Enable automatic execution data pruning |
| `N8N_EXECUTIONS_DATA_PRUNE` | `true` | Enable pruning of old execution data |
| `N8N_EXECUTIONS_DATA_MAX_AGE` | `1` | Max age in hours before pruning (set to minimum) |

### Example Docker Run Command (RHEL/Linux)

```bash
docker run -d \
  --name n8n-waypoint \
  -p 5678:5678 \
  -e N8N_EXECUTIONS_DATA_SAVE_ON_ERROR=none \
  -e N8N_EXECUTIONS_DATA_SAVE_ON_SUCCESS=none \
  -e N8N_EXECUTIONS_DATA_SAVE_ON_PROGRESS=false \
  -e N8N_EXECUTIONS_DATA_SAVE_MANUAL_EXECUTIONS=false \
  -e N8N_EXECUTIONS_PROCESS_DATA_PRUNING=true \
  -e N8N_EXECUTIONS_DATA_PRUNE=true \
  -e N8N_EXECUTIONS_DATA_MAX_AGE=1 \
  -e GENERIC_TIMEZONE=UTC \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n:latest
```

### Example Docker Compose (RHEL/Linux)

```yaml
version: '3.8'
services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n-waypoint
    ports:
      - "5678:5678"
    environment:
      - N8N_EXECUTIONS_DATA_SAVE_ON_ERROR=none
      - N8N_EXECUTIONS_DATA_SAVE_ON_SUCCESS=none
      - N8N_EXECUTIONS_DATA_SAVE_ON_PROGRESS=false
      - N8N_EXECUTIONS_DATA_SAVE_MANUAL_EXECUTIONS=false
      - N8N_EXECUTIONS_PROCESS_DATA_PRUNING=true
      - N8N_EXECUTIONS_DATA_PRUNE=true
      - N8N_EXECUTIONS_DATA_MAX_AGE=1
      - GENERIC_TIMEZONE=UTC
    volumes:
      - n8n_data:/home/node/.n8n
    restart: unless-stopped

volumes:
  n8n_data:
```

## Setup

1. **Start n8n via Docker** using one of the configurations above.

2. **Configure n8n credentials**:
   - **Gmail OAuth2**: Create credential with Gmail + Google Tasks scopes:
     - `https://www.googleapis.com/auth/gmail.readonly`
     - `https://www.googleapis.com/auth/gmail.modify`
     - `https://www.googleapis.com/auth/tasks`
   - **LLM provider credentials**: Add OpenAI API key or compatible provider credentials for AI Agent node.

3. **Import workflow**: Import `workflows/waypoint-email-to-tasks.json` into n8n.

4. **Workflow nodes** (in processing order):
   - Gmail Trigger - Unread Inbox (poll unread inbox; exclude "WAYPOINT-Processed" label)
   - Gmail Message Detail (retrieve full message body + attachments)
   - Cap Input Size (Code node to enforce body/attachment limits)
   - AI Agent - Noise Filter (classify Noise vs Signal)
   - Route Noise (IF node to skip noise emails)
   - AI Agent - Intent & Extraction (classify intent + extract waypoints)
   - Validate Task Card (Code node to validate LLM output against schema)
   - Transform Due Date (convert ISO 8601 to RFC 3339)
   - Google Tasks - Check Duplicate (lookup by idempotency key)
   - Google Tasks - Create (create task with mapped fields)
   - Gmail - Apply Label (apply "WAYPOINT-Processed" label)

5. **Create Error Trigger workflow**: Import `workflows/waypoint-error-trigger.json` to log non-PII metadata per [contracts/error-event.schema.json](contracts/error-event.schema.json).

## Validation

- Run workflow on a small batch of test emails.
- Confirm Task Cards conform to [contracts/task-card.schema.json](contracts/task-card.schema.json).
- Verify Google Tasks entries have correct title, notes, and due date (RFC 3339 when present).
- Confirm emails are labeled "WAYPOINT-Processed" and do not reappear in subsequent runs.

## Testing & Compliance

- See [testing.md](testing.md) for workflow execution test checklist.
- See [compliance.md](compliance.md) for stateless privacy audit checklist.
