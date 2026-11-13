# 0008. The model is chosen per chat, by an LLM profile

- Status: Accepted
- Date: 2025-08-07

## Context

The backend's LLM call could already reach either OpenAI or Amazon Bedrock, picking the client
from the model id, and LLM profiles stored a model id with its prompts in PostgreSQL. Models
differ in price, speed and quality, and new ones appear every few months, so one model for
every chat would either overpay for simple questions or underserve hard ones.

## Decision

- Each chat names an LLM profile (`records.llm_profile_id`), and the profile names the model.
- The model id selects the provider: a `gpt-*` id calls the OpenAI API, any other id calls
  Bedrock (the Claude models, through the EU cross-region inference profiles).
- The catalog of profiles and their prices is data, seeded by `llm-catalog.sql`, not code.

## Consequences

- Adding or retiring a model is mostly rows in `llm_profiles` and `llm_pricing`; a model
  missing from the code's table of context limits is treated as a 32,000-token one until it is
  added there, and a Claude model needs Bedrock access enabled in the account first.
- The LLM call carries one code path per provider, each translating the tool definitions and
  the token counts into a shared shape.
- Usage is priced per model (ADR 0004, ADR 0005), so a chat's cost follows the model it uses.
