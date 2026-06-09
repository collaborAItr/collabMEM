# 12) Sync and Trust

Phase E formalizes cloud-sync trust boundaries for collabMEM&trade;.

## Conflict policy

When local and cloud rows diverge, apply this order:

1. **User edit wins** (`userEdit` / explicit user decision fields).
2. **Higher consensus count wins** (more producer agreement).
3. **Recency wins** (`updatedAt` fallback).

## Tombstones

All primary memory tables now support `deleted_at` tombstones. Scope deletes mark rows tombstoned and emit `mem_sync_events` records rather than hard-deleting immediately.

## Durable retry queue

Extraction retries are persisted in `mem_extraction_retry_queue` and processed by a cron-driven worker. Inspector activity surfaces failed/retrying/recovered states for operator transparency.

## Encryption gating

Default policy for encrypted local memory is **skip cloud push for encrypted tables**. This keeps local-first privacy guarantees while allowing non-sensitive operational metadata sync.
