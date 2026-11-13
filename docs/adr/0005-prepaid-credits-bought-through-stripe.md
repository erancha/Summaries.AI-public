# 0005. Prepaid credits bought through Stripe

- Status: Accepted
- Date: 2026-01-17

## Context

LLM calls cost real money per token, and each user is a tenant of their own (ADR 0002), so each
user must pay for their own usage. A subscription would not track usage that varies by model
and by chat, and the app should never hold card details.

## Decision

- A tenant holds a credit balance in dollars; a new tenant starts with $1.
- Credit is bought in fixed packages through Stripe Payment Intents. The browser confirms the
  card with Stripe.js, and Stripe's `payment_intent.succeeded` webhook adds the credit.
- The remaining credit is the balance minus the summed cost of the tenant's usage rows.

## Consequences

- Card data never reaches the app; the webhook, not the browser, grants credit.
- Usage of any model is charged at its own price with no plan changes.
- The balance is checked by the browser before sending; a backend check does not exist.
