# 0004. Usage metrics flow through Kafka to a consumer of their own

- Status: Accepted
- Date: 2025-08-18

## Context

Every LLM call has a cost that the user's credit pays for, so its token counts must be recorded
reliably. Writing them inside the request would add a database write and a pricing lookup to
every chat turn, and a failure there would fail the turn that the user already received.

## Decision

- The Lambdas publish each call's token counts and LLM profile to the
  `Summaries.AI-usage` Kafka topic and return.
- A separate consumer, `backend/usage-service`, reads the topic in the `usage-metrics-group`
  consumer group, prices each message from `llm_pricing` and writes a `usage_metrics` row,
  committing the offset only after the write.
- The broker and the consumer run on the ECS cluster's EC2 host (ADR 0001).

## Consequences

- A chat turn does not wait for usage accounting, and a consumer outage delays the balance
  instead of losing usage.
- The balance a user sees lags the call by the consumer's delay.
- The broker is a single node on one host; its address is pinned in the Lambdas' parameters.
