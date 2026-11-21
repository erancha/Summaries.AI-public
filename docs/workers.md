# Workers' responsibilities and flows

The workers are `WebsocketEventsHandler`, `ChatSummarizer`, `SummariesStreamProcessor`,
`BucketProcessor` and the usage service; this document details the steps, limits and retries of
each, behind the entry points that [Architecture](architecture.md#the-entry-points) summarizes.
A worker is a function or process no request waits on: a queue message, a table's stream, a
bucket event or a Kafka topic wakes it. The routes that feed them are in
[API endpoints](api.md).

**Contents:** [`WebsocketEventsHandler`](#websocketeventshandler) ·
[`ChatSummarizer`](#chatsummarizer) · [`SummariesStreamProcessor`](#summariesstreamprocessor) ·
[`BucketProcessor`](#bucketprocessor) · [Usage service](#usage-service)

## `WebsocketEventsHandler`

`messagesToClientsQ` (with a dead-letter queue after 5 receives) carries the messages of the
connect, disconnect and billing functions to `WebsocketEventsHandler`
(`websocketEventsHandler.ts`). It pushes each message to its target connections, starts SES
verification of a new user's address, sends an email when a message asks for one, publishes
each data change it carries to `<stack>-commands-event-bus`, and invalidates a CloudFront path
when asked. The bus has a rule only when the `TargetLambdaArn` parameter names a target; the
deploy leaves it empty.

## `ChatSummarizer`

`chatSummarizerQ.fifo` (with a FIFO dead-letter queue after 3 receives) feeds `ChatSummarizer`
(`chat-summarizer/chatSummarizer.ts`) one message at a time, with a reserved concurrency of 20.
After each record or subrecord it asks the summarizer profile (`ChatSummarizerLLMProfileId`)
for a summary that folds in the chat's previous one, writes it to DynamoDB, deletes the
previous summary unless it holds a diagram or code block, and embeds the new one into the
Pinecone summaries index.

## `SummariesStreamProcessor`

`SummariesStreamProcessor` (`documents/summariesStreamProcessor.ts`) reads the Summaries table's
stream: a new summary is rendered to a PDF with Puppeteer and stored in S3; a deleted one has
its PDF deleted and its CloudFront path invalidated. A failing batch is split and retried
twice, then skipped, so one bad summary never blocks the stream.

## `BucketProcessor`

`BucketProcessor` (`bucket-processor/bucketProcessor.ts`) runs on every object created or
removed in the summaries bucket. It writes a `.txt` copy of the text of a PDF or text file; for
a RAG document (`<tenant>/RAG/<file>`) it also cuts the text into chunks, embeds each one and
upserts it into the Pinecone documents index tagged with the tenant. A removed RAG document
loses its `.txt` copy and every chunk.

## Usage service

The usage service (`backend/usage-service/usageTask.js`, an ECS task) consumes the
`Summaries.AI-usage` Kafka topic and writes one `usage_metrics` row per message, with the cost
computed from `llm_pricing` ([Billing](billing.md#how-credit-is-spent)). Its stacks, host and
deploy are in [ECS deployment](ecs-deployment.md).
