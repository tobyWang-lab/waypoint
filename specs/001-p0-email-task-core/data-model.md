# Data Model: P0 Email-to-Task Core Pipeline

## Entities

### Email Signal
**Description**: Email that passed noise filtering.
- **Fields**: `messageId`, `subject`, `sender`, `receivedAt`, `processingStatus`
- **Validation**: `messageId` required; `receivedAt` is ISO 8601 date-time.
- **Relationships**: 1:1 with `Noise Classification`; 1:1 with `Intent Classification`; 1:1 with `Task Card` (when actionable).

### Noise Classification
**Description**: Noise filtering result.
- **Fields**: `messageId`, `noiseCategory` (Advertisement|Newsletter|Notification), `confidence`
- **Validation**: `confidence` in $[0,1]$.
- **Relationships**: Belongs to `Email Signal`.

### Intent Classification
**Description**: Intent result for a signal email.
- **Fields**: `messageId`, `intent` (Actionable|FYI|Forward), `confidence`
- **Validation**: `confidence` in $[0,1]$.
- **Relationships**: Belongs to `Email Signal`.

### Waypoint Extraction
**Description**: Extracted task attributes from actionable email.
- **Fields**: `dueDate` (ISO 8601 date or null), `stakeholders[]`, `actionItems[]`, `extractionConfidence`
- **Validation**: `dueDate` must be ISO 8601 date when present.
- **Relationships**: Belongs to `Email Signal`; feeds `Task Card`.

### Task Card
**Description**: Primary output artifact.
- **Fields**: `id`, `title`, `sourceLink`, `intent`, `dueDate`, `stakeholders[]`, `actionItems[]`, `priority`, `estimatedMinutes`, `category`, `createdAt`, `status`, `statusReason`, `metadata`
- **Validation**: Must conform to Task Card JSON Schema; `status` in {success, partial, failure, skipped}.
- **Relationships**: Derived from `Email Signal` + `Waypoint Extraction`.

## State Transitions

- **Email Signal**: `ingested` → `filtered` → `classified` → `extracted` → `synced` → `labeled`.
- **Task Card status**: `success` | `partial` | `failure` | `skipped` (terminal states per processing result).

## Derived/Computed Fields

- `sourceLink` is derived from Gmail message ID.
- Google Tasks `due` is derived from `dueDate` after RFC 3339 conversion.
- Idempotency key uses Gmail `messageId` stored in Google Tasks `notes`.
