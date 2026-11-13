# 0006. Google sign-in through a shared Cognito user pool

- Status: Accepted
- Date: 2025-07-26

## Context

The repository started as a copy of a skeleton app whose stacks already signed users in through
Cognito. The account held a Cognito user pool of its own stack, `vsdb-cognito`, with Google as
an identity provider and a hosted sign-in domain that Google lists as an authorized redirect.
The app's users are individuals with Google accounts (ADR 0002), and the app should keep no
passwords.

## Decision

- Users sign in with Google only, through the shared `vsdb-cognito` user pool; the app's stack
  adds an app client of its own, whose callbacks are the app's CloudFront URL and
  `http://localhost:3000`.
- The REST API authorizes routes with the user pool. The WebSocket API's `$connect` runs an
  authorizer that verifies the Cognito ID token passed as the `token` query parameter.
- The Cognito `sub` of the signed-in user is their tenant id.

## Consequences

- The app stores no credentials, and a new user needs nothing beyond a Google account.
- The pool's id and domain are stack parameters pinned in `scripts/lib/stack-params.sh`, so the
  app depends on the shared stack being deployed first in the account.
- A change to the pool, such as another identity provider, is made in the shared stack and
  reaches every app that uses it.
