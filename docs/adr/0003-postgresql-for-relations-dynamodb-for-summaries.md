# 0003. PostgreSQL for relations, DynamoDB for summaries

- Status: Accepted
- Date: 2025-08-16

## Context

Chats, their turns, the users they are shared with, LLM profiles, prompts and usage form a
relational model: shared records need a junction table, creating a record with its owner's
access must be atomic, deletes must cascade, and usage analytics aggregate by tenant, profile
and time. Summaries are different: a document per summary, written by an asynchronous worker,
read by chat, with attributes that vary, and each new one must trigger a PDF rendering.

## Decision

- PostgreSQL holds the relational data: tenants, records, subrecords, access, profiles,
  prompts, pricing and usage.
- A DynamoDB table holds the summaries, keyed by id, with an index by record and creation time,
  and a stream that drives the PDF rendering and cleanup.
- Redis caches PostgreSQL reads; Pinecone holds the embeddings.

## Consequences

- Joins, transactions and aggregations stay in SQL, where they are simple.
- The summary pipeline is event-driven through the table's stream, with no polling.
- PostgreSQL and DynamoDB both to operate and back up; a chat's data spans both, linked by
  `record_id`.
- PostgreSQL connections are scarce, so the busy functions' reserved concurrency is sized to the
  server's connection limit.
