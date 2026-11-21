# API endpoints

This document details every route behind the entry points that
[Architecture](architecture.md#the-entry-points) summarizes: who may call it, what it takes and
answers, and its limits. The browser reaches the app through a WebSocket API and a REST API;
another service reaches it through the RAG query API. The work a route hands off runs in the
workers ([Workers' responsibilities and flows](workers.md)).

## WebSocket API

The WebSocket API (`<stack>-WebSocketAPI`) routes by the `action` field of each message.

- `$connect` runs the authorizer (`backend/lambdas/websockets/authorizer/index.js`), which
  verifies the Cognito ID token passed as the `token` query parameter, then `WebsocketConnect`
  (`websockets/connections/connect.ts`).
- `$disconnect` goes to `WebsocketDisconnect` (`websockets/connections/disconnect.ts`).
  Both keep each user's connection ids in Redis and announce the change through the
  messages-to-clients queue.
- `CommandsHandlerRouteKey` is the `action` of every command the browser sends, and goes to
  `WebsocketCommandsHandler` (`websockets/connections/websocketCommandsHandler.ts`). It runs
  each command through the `commandsHandlers` layer: create, read, update and delete of
  records, subrecords, tenants and profiles; searches; the LLM call. It pushes each response to
  the connections of every user the change concerns. Its time limit is 180 seconds and its
  reserved concurrency 80, sized so that with two PostgreSQL connections per instance it stays
  under the server's 200 connections.

## REST API

The REST API (`<stack>-RestAPI`) answers JSON. The Cognito user pool authorizes every route
except `GET /billing/packages` and `POST /billing/webhook`.

- **`SummariesRestFunction`** (`documents/summaries.ts`) — a chat's summaries and files. A
  private chat's summaries answer only its owner (401 for anyone else).
  - `POST /summaries` — creates a summary of the chat's last turn from its `recordId` (201).
  - `GET /summaries` — lists the summaries of the chat named by the `recordId` query parameter.
  - `GET /summaries/{summaryId}` — one summary, named by the `summaryId` query parameter rather
    than the path (400 without it); 404 when there is none.
  - `GET /summaries/{summaryId}/pdf` — a presigned URL of the summary's PDF.
  - `DELETE /summaries/{summaryId}` — deletes one of the caller's summaries.
  - `POST /upload` — stores a file attached to a chat in the summaries bucket; `recordId`,
    `fileName` and `fileType` are required query parameters (400 without them).
- **`DocumentsRestFunction`** (`documents/documents.ts`) — the tenant's RAG documents.
  - `POST /rag/upload` — stores a document under the tenant's RAG folder; only `.pdf`, `.txt`,
    `.md`, `.yml`, `.yaml` and `.ps1` files are accepted (400 for any other).
  - `GET /rag/documents` — lists the tenant's RAG documents.
  - `DELETE /rag/documents/{documentId}` — deletes one of the tenant's documents (403 for
    another tenant's).
- **`BillingFunction`** (`billing/billing.ts`) — credit, detailed in [Billing](billing.md).
  - `GET /billing/packages` — the credit packages on sale.
  - `POST /billing/create-payment-intent` — starts a Stripe purchase of one package.
  - `POST /billing/webhook` — Stripe's notice of a completed payment.
- **`ControlPlaneRedisFunction`** (`control-plane/redis/redis.ts`) — the cache keys; it answers
  only the admin user (`AdminUserId`).
  - `GET /control-plane/redis/keys` — lists cache keys.
  - `DELETE /control-plane/redis/keys` — deletes cache keys.

## RAG query API

The RAG query API (`<stack>-RagQueryAPI`) serves the documents of one tenant, named by the
`RagQueryTenantEmail` stack parameter. Callers authenticate with an API key (`x-api-key`) under
a usage plan. Usage is charged to the configured tenant like its own chat traffic.

- `POST /rag/query` — answers a question from the tenant's documents, with the sources it
  cited. Its integration timeout is 60 seconds.
- `POST /rag/document-url` — presigns one cited document for five minutes; 404 when the tenant
  has no document of that name.
