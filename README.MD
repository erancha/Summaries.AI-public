# Table Of Content

<!-- toc -->

- [Preface](#preface)
  * [Overview](#overview)
    + [SaaS Architecture](#saas-architecture)
  * [Frontend](#frontend)
  * [Backend](#backend)
    + [AI Integration](#ai-integration)
  * [Non-functional attributes](#non-functional-attributes)
    + [Security](#security)
    + [Scalability, Performance and Resiliency](#scalability-performance-and-resiliency)
    + [Deployment](#deployment)
    + [Monitoring and Logging](#monitoring-and-logging)
  * [Data Model](#data-model)
- [Appendix: Hybrid architecture](#appendix-hybrid-architecture)
    + [Option 1: API Gateway + Lambda](#option-1-api-gateway--lambda)
    + [Option 2: ALB + ECS on EC2](#option-2-alb--ecs-on-ec2)
    + [Hybrid architecture Pros and Cons](#hybrid-architecture-pros-and-cons)

<!-- tocstop -->

# Preface

**Summaries.AI** is not just about auto-summaries. It's a [SaaS](#saas) application offering a shared space where people and AI think together. Multiple people chat with an AI (LLM) in a shared context, where everyone can see others’ messages and the AI’s responses in real time.

- **Team up with AI**: Multiple people chat with an LLM in the same space, asking questions, refining answers, and keeping context aligned.
- **Turn chats into knowledge**: Every conversation naturally becomes a structured summary you can save, share, or publish.
- **Find ready knowledge**: Semantic search locates the most relevant summaries by meaning, not just keywords.

**Use cases:**

- A dev team **co-writes a technical summary** before starting a new project.
- Friends **plan a trip**, challenging the AI with different constraints, and end up with a reliable plan.
- Students or professionals **revisit trusted, collaboratively-refined summaries** instead of digging through messy chat histories.

## Overview

![API GW + Lambda](https://lucid.app/publicSegments/view/d9ba40c8-8e3a-419c-92f9-850a9ee95fba/image.jpeg)

Designed with scalability in mind, the application uses:

- AWS serverless computing and storage.
- Redis for distributed caching.
- Global content distribution via CloudFront.
- Monitoring through AWS CloudWatch and X-Ray.
- Real-time updates via WebSockets.

The app features an intuitive mobile-friendly design.

User authentication is securely handled through Google.

### SaaS Architecture

Built as a SaaS solution using the Pool Model (Fully Shared): all tenants share the same infrastructure and database, separated by SaaS tenant unique IDs. In the current design, each user is a tenant (user-as-tenant model), meaning a tenant maps to a single user (no multi-user tenants).

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

- **MCP (Model Context Protocol)**: Embedded two-phase tool calling system where the backend acts as both MCP client and server, enabling LLMs to discover available tools (phase 1), execute them locally, and integrate results into final responses (phase 2).

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

See: [docs/README-data-model.md](docs/README-data-model.md)

# Appendix: Hybrid architecture

The app has an hybrid architecture with a single React frontend that can be configured to connect to one of two backend options at runtime:

This provides deployment flexibility for the same real-time WebSocket app: run on API Gateway + Lambda for serverless elasticity and simpler ops, or on ALB + ECS/EC2 to leverage reserved capacity and predictable steady-state costs.

(Note: Currently only the API Gateway option supports REST endpoints).

### Option 1: API Gateway + Lambda

- Supports both WebSocket connections and REST API calls.
- Serverless, event-driven compute model.

![API GW + Lambda](https://lucid.app/publicSegments/view/d9ba40c8-8e3a-419c-92f9-850a9ee95fba/image.jpeg)

### Option 2: ALB + ECS on EC2

- Supports only WebSocket connections.
- Container-based compute on reserved EC2 instances.

![ALB + ECS](https://lucid.app/publicSegments/view/d6f6a881-f5f2-48d4-a9ff-f57f351a7054/image.jpeg)

### Hybrid architecture Pros and Cons

**Pros**

- **Cost optimization through resource utilization** - Maximizes ROI on existing EC2 reservations while leveraging Lambda's pay-per-request model.
- **Operational flexibility** - Single frontend can be configured for different deployment scenarios, choosing between cost-optimized reserved instances or auto-scaling serverless infrastructure.
- **Built-in redundancy** - WebSocket functionality can fall back between backends, providing resilience against infrastructure failures.

**Cons**

- **Doubled operational overhead** - Requires maintaining two separate backend infrastructures with duplicate monitoring, deployment pipelines, debugging complexity, and expertise in both serverless and container orchestration.
- **Uneven feature support** - ECS backend only supports WebSockets traffic while API Gateway supports both WebSockets and REST, creating design limitations and less-than-ideal routing choices.
