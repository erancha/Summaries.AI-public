# Architecture

This document details the architecture the [README](../README.md) summarizes: every part the
stacks deploy, what the parts share, where the data lives and how the code is laid out. The
credit purchase has its own page, [Billing](billing.md), and the ECS cluster that carries the
usage pipeline has [ECS deployment](ecs-deployment.md).

**Contents:** [One body of code, several entry points](#one-body-of-code-several-entry-points) ·
[The entry points](#the-entry-points) · [Cross-cutting mechanisms](#cross-cutting-mechanisms) ·
[Storage](#storage) · [Codebase structure](#codebase-structure)

## One body of code, several entry points

API Gateway and Lambda serve the whole app: a WebSocket API carries every chat command and
pushes every change to the connected browsers, and REST APIs carry files, summaries, billing
and a service-to-service RAG endpoint. Nothing on the ECS cluster answers the browser; its EC2
host runs the Kafka broker the Lambdas publish usage to, the consumer that turns usage into
cost, and the PostgreSQL and Redis the Lambdas read and write
([ADR 0001](adr/0001-api-gateway-and-lambda-serve-the-app.md)).

![Summaries.AI on AWS](https://lucid.app/publicSegments/view/a2878ac0-1c43-42ab-ab33-088d7a03aa6f/image.jpeg)

The drawing leaves out Stripe and the billing function, Pinecone and the bucket processor that
feeds it, the RAG query API, SES and the commands event bus; it draws the functions in private
subnets, which the template supports but the deploy leaves unset
([Network](#cross-cutting-mechanisms)), and its analytics consumer does not exist. The business
logic lives in Lambda layers the functions share ([Layers](#cross-cutting-mechanisms)), so the
WebSocket handlers, the REST functions and the queue workers run one body of code against the
same stores.

## The entry points

Every function runs Node.js 24 (`nodejs24.x`) with X-Ray tracing on, and writes to a log group
the template declares, `/aws/lambda/<stack>/<logical id>`, kept for `LogRetentionDays` (14).

**Entry points**

- **WebSocket API** (`<stack>-WebSocketAPI`) — routes by the `action` field of each message.
  `$connect` runs the authorizer (`backend/lambdas/websockets/authorizer/index.js`), which
  verifies the Cognito ID token passed as the `token` query parameter. `$connect` and
  `$disconnect` go to `WebsocketConnect` and `WebsocketDisconnect`
  (`websockets/connections/connect.ts`, `disconnect.ts`), which keep each user's connection ids
  in Redis and announce the change through the messages-to-clients queue. Every other message
  is a command for `WebsocketCommandsHandler`.
- **REST API** (`<stack>-RestAPI`) — the Cognito user pool authorizes every route except
  `GET /billing/packages` and `POST /billing/webhook`:
  - `SummariesRestFunction` (`documents/summaries.ts`) — `POST /summaries`,
    `GET /summaries`, `GET /summaries/{summaryId}`, `GET /summaries/{summaryId}/pdf`,
    `DELETE /summaries/{summaryId}`, and `POST /upload` for a file attached to a chat.
  - `DocumentsRestFunction` (`documents/documents.ts`) — `POST /rag/upload`,
    `GET /rag/documents`, `DELETE /rag/documents/{documentId}`: the tenant's RAG documents.
  - `BillingFunction` (`billing/billing.ts`) — `GET /billing/packages`,
    `POST /billing/create-payment-intent`, `POST /billing/webhook` ([Billing](billing.md)).
  - `ControlPlaneRedisFunction` (`control-plane/redis/redis.ts`) — `GET` and `DELETE`
    `/control-plane/redis/keys`, which list and delete cache keys; it answers only the admin
    user (`AdminUserId`).
- **RAG query API** (`<stack>-RagQueryAPI`) — `POST /rag/query` answers a question from the
  documents of one tenant, named by the `RagQueryTenantEmail` stack parameter, and
  `POST /rag/document-url` presigns one cited document for five minutes. Callers authenticate
  with an API key (`x-api-key`) under a usage plan; the query's integration timeout is 60
  seconds. Usage is charged to the configured tenant like its own chat traffic.

**Workers**

- **`WebsocketCommandsHandler`** (`websockets/connections/websocketCommandsHandler.ts`) — runs
  each command through the `commandsHandlers` layer: create, read, update and delete of
  records, subrecords, tenants and profiles; searches; the LLM call. It pushes each response
  to the connections of every user the change concerns. Its time limit is 180 seconds and its
  reserved concurrency 80, sized so that with two PostgreSQL connections per instance it stays
  under the server's 200 connections.
- **`messagesToClientsQ`** (with a dead-letter queue after 5 receives) carries the messages of
  the connect, disconnect and billing functions to **`WebsocketEventsHandler`**
  (`websocketEventsHandler.ts`). It pushes each message to its target connections, starts SES
  verification of a new user's address, sends an email when a message asks for one, publishes
  each data change it carries to `<stack>-commands-event-bus`, and invalidates a CloudFront path
  when asked. The bus has a rule only when the `TargetLambdaArn` parameter names a target; the
  deploy leaves it empty.
- **`chatSummarizerQ.fifo`** (with a FIFO dead-letter queue after 3 receives) feeds
  **`ChatSummarizer`** (`chat-summarizer/chatSummarizer.ts`) one message at a time, with a
  reserved concurrency of 20. After each record or subrecord it asks the summarizer profile
  (`ChatSummarizerLLMProfileId`) for a summary that folds in the chat's previous one, writes it
  to DynamoDB, deletes the previous summary unless it holds a diagram or code block, and embeds
  the new one into the Pinecone summaries index.
- **`SummariesStreamProcessor`** (`documents/summariesStreamProcessor.ts`) — reads the
  Summaries table's stream: a new summary is rendered to a PDF with Puppeteer and stored in S3;
  a deleted one has its PDF deleted and its CloudFront path invalidated. A failing batch is
  split and retried twice, then skipped, so one bad summary never blocks the stream.
- **`BucketProcessor`** (`bucket-processor/bucketProcessor.ts`) — runs on every object created
  or removed in the summaries bucket. It writes a `.txt` copy of the text of a PDF or text file;
  for a RAG document (`<tenant>/RAG/<file>`) it also cuts the text into chunks, embeds each one
  and upserts it into the Pinecone documents index tagged with the tenant. A removed RAG
  document loses its `.txt` copy and every chunk.
- **Usage service** (`backend/usage-service/usageTask.js`, an ECS task) — consumes the
  `Summaries.AI-usage` Kafka topic and writes one `usage_metrics` row per message, with the
  cost computed from `llm_pricing` ([Billing](billing.md#how-credit-is-spent)). Its stacks, host
  and deploy are in [ECS deployment](ecs-deployment.md).

**Hosting and identity**

- **Frontend bucket and CloudFront distribution** — host the React app; `deploy.sh --frontend`
  publishes it.
- **Summaries bucket and its CloudFront distribution** — hold uploaded files, RAG documents and
  summary PDFs. A summary PDF is shared by a presigned S3 URL, valid for 24 hours, whose host
  is rewritten to the distribution's domain. Objects move to Standard-IA after 30 days.
- **Cognito** — the user pool is the shared `vsdb-cognito` stack's, with Google as the only
  identity provider; this stack adds an app client whose callbacks are the CloudFront URL and
  `http://localhost:3000` ([Sign-in](sign-in.md),
  [ADR 0006](adr/0006-google-sign-in-through-a-shared-cognito-user-pool.md)).
- **Secrets Manager secret** `/<app>/app-secrets` (`AppSecretName`) — the OpenAI, Pinecone,
  OpenWeather and Stripe keys the functions read at run time. The main stack owns it; no script
  writes it.

## Cross-cutting mechanisms

- **Tenancy.** One deployment serves every user, and each user is a tenant of their own: every
  row, item, S3 key and Pinecone vector carries the owner's `saas_tenant_id`, the Cognito
  `sub`. No organization signs up with members of its own, which is why the product is a web
  app with prepaid credits rather than a SaaS with tenant administrators
  ([ADR 0002](adr/0002-one-shared-deployment-per-user-tenants.md)).

  ```mermaid
  graph TB
      U1((Tenant A)) --> L
      U2((Tenant B)) --> L
      subgraph APP["One stack, shared by every tenant"]
          L[Lambda functions]
      end
      L -->|"rows keyed by saas_tenant_id"| PG[(PostgreSQL)]
      L -->|"items keyed by saas_tenant_id"| DDB[(DynamoDB)]
      L -->|"keys under the tenant id"| S3[(S3)]
      L -->|"vectors tagged with the tenant"| PC[(Pinecone)]
      subgraph CTRL["Control functions, inside the same app"]
          ID["Identity: Cognito"]
          TEN["Tenants: saas_tenants"]
          MET["Metrics: usage pipeline"]
          BIL["Billing: Stripe credits"]
          ADM["Admin: AdminUserId"]
      end
      L --- CTRL
  ```

- **Layers.** The business logic lives in layers the stack publishes:
  `commandsHandlers` (command routing, the LLM call in `AI/AIProcessor.ts`, the MCP weather
  tools in `AI/MCPServerWeather.ts`, Pinecone), `dbData` (PostgreSQL, DynamoDB and the Redis
  cache), `redisClient`, the WebSocket `connections` helpers, and `documentsUtils` (S3 keys and
  presigned URLs). The shared `vsdb-lambda-layers` stack adds the AWS SDK v3 and the CORS
  headers. SAM uploads a layer under a hash of its content, so a deploy publishes a new
  version only when the layer changed, and `RetentionPolicy: Delete` removes the version it
  replaces.
- **Shared types.** `data-types.ts`, `ws-messages.ts` and `rest-messages.ts` at the repo root
  define the WebSocket and REST contracts. `scripts/lib/transpile.sh` hard-links them into the
  frontend and every backend folder that compiles them, so both sides compile one definition.
- **LLM call.** A chat's LLM profile, a row of `llm_profiles`, names the model: a `gpt-*` id calls
  the OpenAI Chat Completions API, an `eu.anthropic.claude-*` id calls Bedrock's EU cross-region
  inference profile ([ADR 0008](adr/0008-model-chosen-per-chat-by-llm-profile.md)). The call has two
  phases: the model first sees the MCP weather tools and may ask for calls, the backend runs them,
  and the model then writes the answer with their results. Before the call the backend searches the
  tenant's RAG documents and adds the matching chunks (top 10, similarity at least 0.3) to the
  prompt. Each call's token counts go to Kafka. Embeddings use OpenAI `text-embedding-3-small` and
  are not metered.
- **Search.** A chat search is semantic by default, a Pinecone query over the summaries index,
  or a keyword search over a PostgreSQL full-text index of each record's subject, user text and
  the first 2,000 characters of the answer.
- **Caching.** `dbData` caches query results in Redis under `<stack>:db:` keys, capped at
  10,000 keys with least-recently-used eviction kept in a sorted set; `DisableAppCache` turns it
  off. Redis also holds the WebSocket connection registry under `<stack>:websocket:` keys.
- **Configuration.** Every value reaches a function as an environment variable set by the
  template from a stack parameter, and `scripts/lib/stack-params.sh` lists every parameter the
  deploy passes. The feature-branch stack `sum-fb` reuses the main stack's Summaries table,
  summaries bucket and distribution through the `Existing*` parameters. The frontend reads
  `frontend/src/appConfig.json`, which `scripts/lib/app-config.sh` writes from the stack's outputs.
- **Network.** The functions run outside a VPC: the template places them in a private subnet
  only when `ExistingAppPrivateSubnetId` and `ExistingAppSGId` are set, and the deploy leaves
  both empty. They reach PostgreSQL, Redis and Kafka at the ECS host's public DNS name
  (`ECS_HOST_ADDRESS` in `stack-params.sh`).
- **Nothing runs on a clock** in the stack. On the ECS host a loop publishes `docker stats`
  every 10 seconds ([ECS deployment](ecs-deployment.md#what-the-host-runs)).

## Storage

The relational data lives in PostgreSQL and the summaries in DynamoDB
([ADR 0003](adr/0003-postgresql-for-relations-dynamodb-for-summaries.md)). The PostgreSQL
schema is `createTables` in the `dbData` layer's `pgData.ts`, which no deployed code calls (its
call in `dbData.ts` is commented out); `llm-catalog.sql` beside it seeds the LLM profiles and
prices through `scripts/tools/sql-helper.sh`.

**PostgreSQL**, database `sum_db` on the ECS host:

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

**DynamoDB**, table `<stack>-Summaries` (billed per request, point-in-time recovery, stream of
new and old images): `id` (partition key), `record_id`, `subject`, `content` (the markdown a
person reads), `summary` (the text the next summary builds on), `direction` (`ltr` or `rtl`),
`saas_tenant_id`, `created_at`. The `RecordCreatedIndex` index (`record_id`, `created_at`)
lists a chat's summaries in time order. A delete is conditional on the caller's tenant.

**S3**, bucket `<stack>-summaries-<account>`:

- `<tenant>/<record>/summary-<summary>.pdf` — a summary's PDF.
- `<tenant>/<record>/<file>` — a file attached to a chat.
- `<tenant>/RAG/<file>` — a RAG document, with its `.txt` copy beside it.

**Pinecone**: `summaries-index` (summary embeddings, for semantic search) and `documents-index`
(RAG chunks, whose metadata `object_key` names the S3 document), 1,536 dimensions, cosine. The
app connects to them by name and creates neither
([ADR 0007](adr/0007-embeddings-in-pinecone.md)). `scripts/tools/rag-maintenance.sh` reconciles the
documents index with S3, since nothing else records which documents exist.

**Redis** holds only the cache and the connection registry, so losing it loses no data.

## Codebase structure

- `backend/lambdas/` — one folder per function package: `websockets/authorizer`,
  `websockets/connections` (connect, disconnect, commands and events handlers), `documents`
  (summaries and documents REST, the stream processor), `bucket-processor`, `chat-summarizer`,
  `billing`, `control-plane/redis`, `rag-query`.
- `backend/summaries-service/layers/` — the `commandsHandlers`, `dbData` and `redisClient`
  layers; `backend/layers/documents-utils` and `backend/lambdas/websockets/layers/connections`
  hold the rest.
- `backend/usage-service/` — the Kafka consumer and its Dockerfile.
- `backend/tests/` — Node test runner suites for the document ids, the RAG query contract and
  the stream processor.
- `frontend/` — the React app: `src/components/`, `src/redux/` (one folder per slice of
  state), `src/services/` (REST and Stripe clients), `src/locales/` (English and Hebrew).
- `data-types.ts`, `ws-messages.ts`, `rest-messages.ts` — the shared contracts (above).
- `scripts/` — the templates and the operations scripts ([Development](development.md)).
- `docs/` — this document, the topic documents and the ADRs under `docs/adr/`.
