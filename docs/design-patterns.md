# Design patterns

This document details the mechanisms that cut across the entry points
[Architecture](architecture.md) lists: where the shared code lives, how a reply reaches the
browser, how a setting reaches a function and the browser, where the functions run and what
wakes them; then how the browser and the backend keep one contract, what
is cached and who receives email.

**Contents:** [Shared code lives in layers](#shared-code-lives-in-layers) ·
[Replies go through a queue](#replies-go-through-a-queue) ·
[Settings come from the template and the stack's outputs](#settings-come-from-the-template-and-the-stacks-outputs) ·
[The functions run outside a VPC](#the-functions-run-outside-a-vpc) ·
[Nothing runs on a clock](#nothing-runs-on-a-clock) ·
[One definition of each contract](#one-definition-of-each-contract) ·
[Query results are cached in Redis](#query-results-are-cached-in-redis) ·
[Email goes only to verified addresses](#email-goes-only-to-verified-addresses)

## Shared code lives in layers

The business logic lives in Lambda layers the stack publishes, so every function that needs a
rule imports the same code:

- `commandsHandlers` — command routing, the LLM call in `AI/AIProcessor.ts`, the MCP weather
  tools in `AI/MCPServerWeather.ts`, and Pinecone.
- `dbData` — PostgreSQL, DynamoDB and the Redis cache.
- `redisClient`, and the WebSocket `connections` helpers.
- `documentsUtils` — S3 keys and presigned URLs.

The shared `vsdb-lambda-layers` stack adds the AWS SDK v3 and the CORS headers. SAM uploads a
layer under a hash of its content, so a deploy publishes a new version only when the layer
changed, and `RetentionPolicy: Delete` removes the version it replaces.

## Replies go through a queue

A function that answers a browser either pushes to its WebSocket connections itself or queues the
message to `WebsocketEventsHandler`, the events handler
([Workers' responsibilities and flows](workers.md#websocketeventshandler)), which pushes it. The
template gives the API's management address only to the commands handler and the disconnect
function, so those two push themselves; the connect function and the summaries, documents and
billing REST functions queue their replies to `MessagesToClientsQueue`.

## Settings come from the template and the stack's outputs

The functions and the browser each read their settings from one place. Every value reaches a
function as an environment variable set by the template from a stack parameter, and
`scripts/lib/stack-params.sh` lists every parameter the deploy passes. The feature-branch stack
`sum-fb` reuses the main stack's Summaries table, summaries bucket and distribution through the
`Existing*` parameters. The frontend reads `frontend/src/appConfig.json`, which
`scripts/lib/app-config.sh` writes from the stack's outputs.

## The functions run outside a VPC

The template places the functions in a private subnet only when `ExistingAppPrivateSubnetId` and
`ExistingAppSGId` are set, and the deploy leaves both empty. They reach PostgreSQL, Redis and
Kafka at the ECS host's public DNS name (`ECS_HOST_ADDRESS` in `stack-params.sh`).

## Nothing runs on a clock

No schedule wakes any function of the stack: each one runs on a request, a queue message, a
stream record or a bucket event. On the ECS host a loop publishes `docker stats` every 10
seconds ([ECS deployment](ecs-deployment.md#what-the-host-runs)).

## One definition of each contract

The frontend and the backend compile the same three files, `data-types.ts`, `ws-messages.ts` and
`rest-messages.ts` at the repo root, which define the WebSocket and REST contracts.
`scripts/lib/transpile.sh` hard-links them into the frontend and every backend folder that
compiles them, so both sides compile one definition.

## Query results are cached in Redis

`dbData` caches query results in Redis under `<stack>:db:` keys, capped at 10,000 keys with
least-recently-used eviction kept in a sorted set; `DisableAppCache` turns it off. Redis also
holds the WebSocket connection registry under `<stack>:websocket:` keys.

## Email goes only to verified addresses

The events handler sends from one fixed sender address set in the template, and the SES sandbox
sends only to verified addresses. When a user with no tenant yet connects, the message
`WebsocketConnect` queues names the user's address, and the events handler asks SES to verify an
address SES does not know yet; for a known address it reads whether SES has verified it. The
message it then pushes to the browser says when verification is pending.
