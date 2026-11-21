# Data model

This document details the stores that [Architecture](architecture.md#storage) summarizes: every
PostgreSQL table with its columns and indexes, the DynamoDB table, the layout of the S3 bucket,
the Pinecone indexes and what Redis holds. Every row, item, key and vector a tenant owns carries
the tenant's `saas_tenant_id`
([Every user is a tenant of their own](architecture.md#every-user-is-a-tenant-of-their-own)).

**Contents:** [PostgreSQL](#postgresql) · [DynamoDB](#dynamodb) · [S3](#s3) ·
[Pinecone](#pinecone) · [Redis](#redis)

## PostgreSQL

The database `sum_db` runs on the ECS host. Its schema is `createTables` in the `dbData` layer's
`pgData.ts`, which no deployed code calls (its call in `dbData.ts` is commented out);
`llm-catalog.sql` beside it seeds the LLM profiles and prices through
`scripts/tools/sql-helper.sh`.

- `llm_profiles` — `profile_id` (UUID key), `model_id`, `description`, `level`, `disabled`,
  `priority` (display order), `notes`, timestamps. A partial index orders the enabled ones.
- `llm_prompts` — `prompt_id`, `name`, `user_prompt`, `system_prompt`, timestamps.
- `saas_tenants` — `saas_tenant_id` (UUID, the Cognito `sub`), `disabled`, `credit` (whole
  dollars, integer, default 0; a new tenant starts at 1), `email`, `name`, `phone`, `address`,
  `israeli_id` (8 or 9 digits, checked), timestamps.
- `records` — one chat: `record_id`, `user_text`, `ai_text`, `subject`, `llm_profile_id` and
  `llm_prompt_id` (set to null when the profile or prompt is deleted), `access_level` (0
  private, 1 restricted, 2 public), `is_archived`, `saas_tenant_id` (deleted with its tenant),
  timestamps. Indexes cover a tenant's live records by update time, the public and the
  linkable (public or restricted) records, archived records, and the full-text search.
- `subrecords` — a chat's follow-up turns: `subrecord_id`, `record_id` (deleted with its
  record), `user_text`, `ai_text` (null when `non_ai`), `non_ai`, `is_consolidated` (the turn
  that consolidated the chat so far), `saas_tenant_id`, `created_at`.
- `records_user_access` — who a record is shared with: `record_id`, `user_id`,
  `shared_by_user_id`, `is_pending`, `created_at`; one row per record and user.
- `usage_metrics` — one LLM call: `usage_id`, `saas_tenant_id`, `llm_profile_id`,
  `prompt_tokens`, `completion_tokens`, `reasoning_tokens`, `cost` (dollars, six decimals),
  `created_at`; indexed by tenant and by profile, each by time.
- `llm_pricing` — `model_id` (key), `input_cost_per_million`, `output_cost_per_million`,
  timestamps.

Record creation writes the record and its owner's access row in one transaction. Archiving sets
`is_archived` and deletes nothing.

## DynamoDB

The table `<stack>-Summaries` is billed per request, with point-in-time recovery and a stream of
new and old images. Its attributes: `id` (partition key), `record_id`, `subject`, `content` (the
markdown a person reads), `summary` (the text the next summary builds on), `direction` (`ltr` or
`rtl`), `saas_tenant_id`, `created_at`. The `RecordCreatedIndex` index (`record_id`, `created_at`)
lists a chat's summaries in time order. A delete is conditional on the caller's tenant.

## S3

The bucket `<stack>-summaries-<account>` holds every file, behind a CloudFront distribution of its
own:

- `<tenant>/<record>/summary-<summary>.pdf` — a summary's PDF.
- `<tenant>/<record>/<file>` — a file attached to a chat.
- `<tenant>/RAG/<file>` — a RAG document, with its `.txt` copy beside it.

A summary PDF is shared by a presigned S3 URL, valid for 24 hours, whose host is rewritten to the
distribution's domain. Objects move to Standard-IA after 30 days.

## Pinecone

Two indexes: `summaries-index` (summary embeddings, for semantic search) and `documents-index`
(RAG chunks, whose metadata `object_key` names the S3 document), 1,536 dimensions, cosine. The
app connects to them by name and creates neither
(ADR 0007). `scripts/tools/rag-maintenance.sh` reconciles the
documents index with S3, since nothing else records which documents exist.

## Redis

Redis holds only the query cache and the WebSocket connection registry
([Query results are cached in Redis](design-patterns.md#query-results-are-cached-in-redis)), so
losing it loses no data.
