# Table Of Content

<!-- toc -->

- [Preface](#preface)
- [High-Level Design (HLD)](#high-level-design-hld)
  - [Hybrid architecture](#hybrid-architecture)
    - [Option 1: API Gateway + Lambda](#option-1-api-gateway--lambda)
    - [Option 2: ALB + ECS on EC2](#option-2-alb--ecs-on-ec2)
  - [Overview](#overview)
  - [Frontend](#frontend)
  - [Backend](#backend)
    - [AI Integration](#ai-integration)
    - [Hybrid architecture](#hybrid-architecture-1)
  - [Non-functional attributes](#non-functional-attributes)
    - [Security](#security)
    - [Scalability, Performance and Resiliency](#scalability-performance-and-resiliency)
    - [Deployment](#deployment)
    - [Monitoring and Logging](#monitoring-and-logging)
  - [Data Model](#data-model)

<!-- tocstop -->

# Preface

**Summaries.AI** is not just about auto-summaries. It’s a shared space where people and AI think together.

- **Team up with AI**: Multiple people chat with an LLM in the same space, asking questions, refining answers, and keeping context aligned.
- **Turn chats into knowledge**: Every conversation naturally becomes a structured summary you can save, share, or publish.
- **Find ready knowledge**: Semantic search locates the most relevant summaries by meaning, not just keywords.

**Use cases:**

- A dev team **co-writes a technical summary** before starting a new project.
- Friends **plan a trip**, challenging the AI with different constraints, and end up with a reliable plan.
- Students or professionals **revisit trusted, collaboratively-refined summaries** instead of digging through messy chat histories.

# High-Level Design (HLD)

## [Hybrid architecture](#hybrid-architecture-pros-and-cons)

The app has an hybrid architecture with a single React frontend that can be configured to connect to one of two backend options at runtime:

### Option 1: API Gateway + Lambda

- Supports both WebSocket connections and REST API calls.
- Serverless, event-driven compute model.

![API GW + Lambda](https://lucid.app/publicSegments/view/78f184da-16f0-4381-8ba3-137438d4f089/image.jpeg)

### Option 2: ALB + ECS on EC2

- Supports only WebSocket connections.
- Container-based compute on reserved EC2 instances.

![ALB + ECS](https://lucid.app/publicSegments/view/d6f6a881-f5f2-48d4-a9ff-f57f351a7054/image.jpeg)

## Overview

Designed with scalability in mind, the application uses:

- AWS serverless computing and storage.
- Redis for distributed caching.
- Global content distribution via CloudFront.
- Monitoring through AWS CloudWatch and X-Ray.
- Real-time updates via WebSockets.

Built as a SaaS solution using the Pool Model (Fully Shared), where all tenants share the same infrastructure and database with data separation through SaaS tenant unique IDs.

The app features an intuitive mobile-friendly design.

User authentication is securely handled through Google.

## Frontend

- Single Page Application (SPA) developed with React.
- Hosted on AWS S3 (pre-configured to one of the [hybrid backend](#hybrid-architecture) options).
- Delivered globally via **AWS CloudFront**.
- Technology stack: **React**, **Redux**, **TypeScript**.

## Backend

- Data is persisted in S3, PostgreSQL and DynamoDB, with Redis for improved read performance.
- SQS queues handle WebSocket notifications asynchronously, allowing Lambda functions in private subnets to process data requests immediately without waiting for notification delivery, publish to EventBridge, and handle CloudFront cache invalidation.

### AI Integration

- **LLM selection**: Users can choose from multiple leading LLMs (e.g., GPT, Claude, etc.), ensuring flexibility for different use cases.

- **Embeddings and semantic memory**: Chat summaries are embedded and stored in **Pinecone vector DB**. This enables semantic retrieval of summaries, allowing users to locate relevant knowledge by meaning rather than keywords.

- **Knowledge reuse**: Summaries are injected back into the shared context, ensuring continuity across sessions and improving the quality of group outputs.

- **RAG (Retrieval-Augmented Generation)**: Users can upload PDF documents, which are automatically processed and chunked. When creating new chats or chat message, the system performs semantic search across uploaded documents to find relevant content and automatically includes it as context for the AI, enabling responses that incorporate knowledge from uploaded documents without manual reference.

### [Hybrid architecture](#hybrid-architecture) Pros and Cons

**Pros**

- **Cost optimization through resource utilization** - Maximizes ROI on existing EC2 reservations while leveraging Lambda's pay-per-request model.
- **Operational flexibility** - Single frontend can be configured for different deployment scenarios, choosing between cost-optimized reserved instances or auto-scaling serverless infrastructure.
- **Built-in redundancy** - WebSocket functionality can fall back between backends, providing resilience against infrastructure failures.

**Cons**

- **Doubled operational overhead** - Requires maintaining two separate backend infrastructures with duplicate monitoring, deployment pipelines, debugging complexity, and expertise in both serverless and container orchestration.
- **Uneven feature support** - ECS backend only supports WebSockets traffic while API Gateway supports both WebSockets and REST, creating design limitations and less-than-ideal routing choices.

## Non-functional attributes

### Security

- Data in transit is encrypted with HTTPS.
- User authentication via AWS Cognito with **Google integration**.
- Lambda functions and Redis are in **private subnet**s.
- IAM roles follow the least privilege principle.
- Summaries in S3 are shared via presigned URLs, which are configured with an expiration time (e.g., 1 day) to limit exposure.

### Scalability, Performance and Resiliency

- Serverless architecture enables automatic scaling.
- Redis enhances the scalability of read operations.
- CloudFront provides low-latency content delivery.

### Deployment

- Uses AWS SAM (Serverless Application Model) for deployment.
- Infrastructure is defined with CloudFormation templates.
- Deploy with a single command: `sam build` and `sam deploy`.
- The app is available online at https://d2k7eoph8u0dj4.cloudfront.net.

- **SaaS Capabilities**:
  - Multi-tenant architecture using Pool Model (Fully Shared).
  - Self-service onboarding for role1s.
  - Centralized cloud-based delivery.
  - Automatic updates and maintenance.
  - Note: Currently free to use (no subscription model implemented).

### Monitoring and Logging

- **Monitoring** and **logging** via AWS CloudWatch and X-Ray.

**AWS X-Ray**

- **Purpose**: AWS X-Ray is used to trace requests as they travel through the application, providing insights into performance bottlenecks and service dependencies.
- **Impact on Production Performance**: Minimal impact when sampling is enabled. Sampling ensures that only a subset of requests are traced, reducing overhead.
- **Benefits**: Helps in identifying latency issues, debugging errors, and understanding the application's behavior under load.

## Data Model

**1. LLM profiles Table**:

- **profile_id** (PK) - Unique identifier for each profile (UUID).
- **description** - String. Profile's description.
- **modelId** - String. Model id (e.g. 'gpt-4.1-nano').
- **userPrompt** - String. User prompt.
- **systemPrompt** - String. System prompt.
- **created_at** - Timestamp when the profile was created.
- **updated_at** - Timestamp when the profile was last modified.

**2. SaaS Tenants Table**:

- **saas_tenant_id** (PK) - Identifier for SaaS multi-tenancy.
- **name**
- **email**
- **phone**
- **address**
- **disabled** - If disabled, the role1 account is no longer active.

**3. Chats Table**:

- **chat_id** (PK) - Unique identifier for each chat (UUID).
- **llm_profile_id** (FK) - Unique identifier of the LLM profile used by ths chat (UUID).
- **user_text** - String. User entered text.
- **ai_text** - String. AI generated text.
- **created_at** - Timestamp when the chat was created.
- **updated_at** - Timestamp when the chat was last modified.
- **saas_tenant_id** (Partition Key) - Identifier for SaaS multi-tenancy.

**4. ChatMessagess Table**:

- **messages_id** (PK) - Unique identifier for each chat message (UUID).
- **chat_id** - Identifier for the chat associated with the chat messages.
- **created_at** - Date when the chat message was made.
- **user_text** - String. Chat message's user entered text.
- **ai_text** - String. Chat message's AI generated text.
- **confirmed_at**: - Date of confirmation.
- **saas_tenant_id** (Partition Key) - Identifier for SaaS multi-tenancy.

**5. Chats_user_access Table**:

- **id** (PK) - Auto-incrementing primary key.
- **chat_id** (FK) - References Chats(chat_id), cascades on delete.
- **user_id** (FK) - References saas_tenants(saas_tenant_id), user who has access to the chat.
- **shared_by_user_id** (FK) - References saas_tenants(saas_tenant_id), user who shared the chat.
- **created_at** - Timestamp when access was granted (defaults to now()).
- **Unique constraint** - Ensures one access chat per (chat_id, user_id) pair.

**6. Summaries Table**:

- **id** (Partition Key) - Unique identifier for each summary (UUID).
- **chat_id** - Identifier for the chat associated with the summary.
- **pdf_url** - URL of the pdf in S3.
- **created_at** - Timestamp when the summary was created.
- **saas_tenant_id** (GSI Partition Key) - Identifier for SaaS multi-tenancy.

**Example of Relationships**:

- A user can have multiple chats.
- A chat can have multiple messages.
