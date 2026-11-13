# 0001. API Gateway and Lambda serve the app

- Status: Accepted
- Date: 2026-11-10

## Context

The repository started from a skeleton that offered two backends side by side: API Gateway with
Lambda, and an Application Load Balancer in front of ECS services. The ECS stacks could still
deploy an ALB, a WebSocket-serving summaries service and a scraper, and the frontend could be
switched between the two. Nothing ran that way: the live cluster never had a load balancer, and
the frontend always shipped the API Gateway URLs. The cluster's EC2 host was meanwhile in real
use, as the host of the Kafka broker the usage pipeline needs and of the account's databases.

## Decision

- A WebSocket API and REST APIs on API Gateway, each route served by a Lambda function,
  are the only way into the app.
- The ECS cluster stays, as the host of the usage pipeline: one EC2 host running the Kafka
  broker and container monitoring, and the usage consumer as its service, which takes no inbound
  traffic.
- The ALB, its listeners and target groups, and the ECS services that served the browser are
  removed from the templates and scripts.

## Consequences

- One code path serves the browser, so a feature is built and tested once.
- Traffic scales and is billed per request; an idle app costs only its storage and the EC2 host.
- Long work runs inside the Lambda limits: the commands handler has 180 seconds, and the
  REST APIs' integrations stop at 29 seconds except the RAG query's 60.
- The EC2 host is a single instance without an Elastic IP, so a replaced host means updating
  its address in `stack-params.sh` and `deploy-ecs.sh`.
