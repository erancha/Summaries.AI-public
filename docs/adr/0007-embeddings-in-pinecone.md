# 0007. Embeddings live in Pinecone

- Status: Accepted
- Date: 2025-07-28

## Context

The app's search was to find chats by meaning, which needs an embedding of each summary and a
nearest-neighbor query over them. DynamoDB, which held the summaries, has no vector search.
Pinecone offers a managed vector index with a Node.js client, reachable over HTTPS from a
Lambda, so no vector store had to be run by the app.

## Decision

- Embeddings are stored in Pinecone, in indexes the app connects to by name and never creates:
  `summaries-index` for chat summaries, and `documents-index` for chunks of the documents
  users upload for RAG.
- The vector id or metadata ties each embedding back to its record or S3 document, and the
  document chunks carry the tenant id, by which every RAG query filters.
- A summary search maps its matches to record ids and reads those records from PostgreSQL, so
  the access rules of ADR 0002 decide what the user sees.

## Consequences

- No vector extension or index to operate on the database host.
- Another external service with its own API key in the app secret, and a network call per
  search and per write.
- Nothing but Pinecone records which documents were embedded, so `scripts/tools/rag-maintenance.sh`
  reconciles the documents index with S3.
- The indexes' dimension (1,536) is fixed to the embedding model; a model with another dimension
  needs new indexes.
