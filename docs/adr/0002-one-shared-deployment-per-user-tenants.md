# 0002. One shared deployment, each user a tenant of their own

- Status: Accepted
- Date: 2025-07-26

## Context

The app lets people chat with an LLM in a shared context and keep the summaries. Its users are
individuals who sign in with Google, not organizations that bring their own members and admins.
A deployment per customer would multiply the stacks, databases and costs for users who each
hold a few chats.

## Decision

- One stack per branch serves every user, with shared functions, databases, bucket and indexes
  (all tenants in one pool).
- Each user is a tenant: the `saas_tenants` row keyed by the Cognito `sub`, whose id every
  record, subrecord, summary, usage row, S3 key prefix and Pinecone vector carries.
- A read of a user's own data filters by the caller's tenant id. Another user sees a record
  only when it is public or shared with them by a row in `records_user_access`.

## Consequences

- Adding a user costs nothing beyond their data, and a deploy updates everyone at once.
- Isolation rests on the code filtering by tenant everywhere; no database or network boundary
  separates tenants.
- With no organization layer, the product is a web app with prepaid credits per user, not a
  SaaS with tenant administrators; the admin is one user id, `AdminUserId`.
