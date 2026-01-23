# Research Notes: P0 Email-to-Task Core Pipeline

## Decision 1: Direct Google Tasks delivery (no Dashboard)
- **Decision**: Task Cards are delivered directly to Google Tasks; no webhook/Dashboard intermediary.
- **Rationale**: Aligns with MVP scope and Constitution Principle IV; reduces stateful surface.
- **Alternatives considered**: Dashboard webhook delivery with sequential sync; rejected as out of scope and adds persistence.

## Decision 2: Stateless processing enforcement
- **Decision**: Disable execution data saving for all runs; log only non-PII metadata via Error Trigger workflow.
- **Rationale**: Ensures compliance with Constitution Principle V even on failures.
- **Alternatives considered**: Error-only data retention with short TTL; rejected due to residual PII risk.

## Decision 3: Duplicate prevention
- **Decision**: Use Gmail label plus deterministic idempotency key (Gmail message ID) stored in Google Tasks notes.
- **Rationale**: Prevents duplicates when labeling fails, without external state.
- **Alternatives considered**: External dedupe store; rejected due to added state and infrastructure.

## Decision 4: Oversized input handling for LLM
- **Decision**: Cap LLM input size and attachment bytes; skip oversized items with partial status.
- **Rationale**: Works within n8n AI Agent limits and avoids timeouts.
- **Alternatives considered**: Chunking into multiple LLM calls; rejected to minimize complexity in MVP.

## Decision 5: Date conversion before Google Tasks sync
- **Decision**: Explicitly convert ISO 8601 dates to RFC 3339; omit due date on conversion failure.
- **Rationale**: Prevents Google Tasks API rejections and keeps data flow testable.
- **Alternatives considered**: Rely on API coercion; rejected due to inconsistent behavior.