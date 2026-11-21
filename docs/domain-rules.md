# Domain rules

This document details the LLM call and the search that [Architecture](architecture.md)
summarizes: how a chat question is answered and how a search runs. The routes are in
[API endpoints](api.md), the workers in [Workers' responsibilities and flows](workers.md), and
how credit is bought and spent in [Billing](billing.md).

## How a chat question is answered

A chat's LLM profile, a row of `llm_profiles`, names the model: a `gpt-*` id calls the OpenAI
Chat Completions API, an `eu.anthropic.claude-*` id calls Bedrock's EU cross-region inference
profile (ADR 0008).

- **Before the call** the backend searches the tenant's RAG documents and adds the matching
  chunks (top 10, similarity at least 0.3) to the prompt.
- **The call has two phases:** the model first sees the MCP weather tools and may ask for calls,
  the backend runs them, and the model then writes the answer with their results.
- **After the call** its token counts go to Kafka. Embeddings use OpenAI
  `text-embedding-3-small` and are not metered.

## Search

A chat search is semantic by default, a Pinecone query over the summaries index, or a keyword
search over a PostgreSQL full-text index of each record's subject, user text and the first 2,000
characters of the answer.
