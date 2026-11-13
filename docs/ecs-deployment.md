# ECS Deployment

Behind [Architecture](architecture.md) and the [README](../README.md)'s Getting started, this page
gives the deploy procedure of the usage-metrics pipeline's ECS stacks and the one list of what their
EC2 host runs. The Lambdas publish every request's LLM usage to a Kafka broker on that host, and the
usage service, an ECS service on the cluster, writes it to PostgreSQL. The same instance also
carries the monitoring of its containers and the Postgres and Redis of every app in the account. The
page then covers the order the stacks deploy in and the host-bound address that needs attention
after a new host comes up. The app itself is served by API Gateway and Lambda only; nothing on the
cluster answers the frontend.

**Contents:** [The stacks](#the-stacks) · [What the host runs](#what-the-host-runs) ·
[Prerequisites](#prerequisites) · [Deploying](#deploying) ·
[After a new host: the Kafka broker address](#after-a-new-host-the-kafka-broker-address) ·
[Undeploying](#undeploying)

## The stacks

Both stacks sit beside the branch's API Gateway stack and take its parameters: `main` targets
production (`sum`), any other branch `sum-fb`. `deploy-ecs.sh` has no further guard, so on `main`
every command below acts on production.

| Stack | Template | Holds |
|---|---|---|
| `<stack>-ecs-c` | `scripts/template-ecs-c.yaml` | the cluster and its EC2 host |
| `<stack>-ecs-s-<prefix>` | `scripts/template-ecs-s.yaml` | one service on that cluster |

`<prefix>` is the service folder's name up to its first dash: `usage-service` deploys as
`sum-ecs-s-usage`.

**The cluster stack** creates the ECS cluster, a `t4g.small` host (ECS-optimized Amazon Linux 2,
arm64) in an Auto Scaling group pinned to one instance across the `vsdb` VPC's two public subnets,
and a capacity provider with managed scaling off. The host's first boot joins the cluster and
starts the Compose project of the [next section](#what-the-host-runs).

**A service stack** creates a log group (`/aws/ecs/<stack>`), a task definition (`awsvpc`, 256 CPU
units, 512 MiB, the image the deploy just pushed), a service of one task, and a scaling policy
that keeps 1 to 3 tasks around 70% CPU. The task listens on no port: it consumes Kafka and writes
to Postgres. Its database endpoints, passwords and Kafka address are copied from
`scripts/lib/stack-params.sh`, so the Lambdas and the task share one set of values.

## What the host runs

The account has one EC2 instance, this stack's host, and every workload below shares its memory
and its root disk. This table is the one list of them; the other documents and the scripts'
comments point here rather than repeat it.

| Domain | Workload | Port | Defined by | Used by |
|---|---|---|---|---|
| Compute | the ECS agent and the tasks of the service stacks, today the usage service | | this stack and `template-ecs-s.yaml` | the usage pipeline |
| Messaging | Redpanda, the Kafka broker; Redpanda Console | 19092; 8080 | this stack's Compose project `local`, written by the user data to `/usr/local/docker-compose.yml` | the Lambdas publish usage metrics, the usage service consumes them |
| Observability | a loop publishing `docker stats` of every container to the `docker-stats-ecs-kafka` topic every 10 seconds; the `kafka-to-influxdb` consumer (the `docker-stats-grafana` image); InfluxDB; Grafana | 8086; 3100 | the same project, plus `send-stats-to-kafka.sh` from the user data | the container dashboard below |
| Databases | Postgres 15 and Redis 7 | 5432, 6379 | vpc-subnets-sgs, as its Compose project `databases` under its security group `MyDbSG`, which this host carries beside its own (`ExistingDbSGId`) | every app in the account, this one included: its Lambdas by the host's public name, its tasks by the private one |

The Compose volumes live on the instance: the Kafka topics go with the stack, while the root
disk, which also holds the databases' volumes, is kept on termination. The host's address is the
one value its clients carry, so a replacement host means the
[procedure below](#after-a-new-host-the-kafka-broker-address) for this app and a repoint of the
databases' other clients by vpc-subnets-sgs.

**The container dashboard.** Grafana on the host, `http://<host public DNS name>:3100`, user
`admin` with the password set in the cluster template's user data, opens on a dashboard of
`docker stats` for every container on the host, the databases included. At first boot the user
data downloads the dashboard and its InfluxDB datasource from the consumer image's repository,
`erancha/docker-stats-grafana`, into `/usr/local/grafana/provisioning`, which the Compose file
mounts into Grafana. A dashboard change is made in that repository; a new host picks it up at
first boot, and the running host by copying the files there and restarting the `grafana`
container. The InfluxDB data lives in a Compose volume on the host.

## Prerequisites

The [prerequisites of a backend deploy](development.md#prerequisites), plus:

- Docker and `npm` for the image build (the script starts Docker Desktop when the daemon is down).

## Deploying

Cluster first, then the usage service, from anywhere in the repo.

```bash
./scripts/deploy-ecs.sh                                    # 1. the cluster and its host
./scripts/deploy-ecs.sh --service usage-service            # 2. the Kafka usage consumer
```

Every deploy validates and builds its template with SAM, then runs `sam deploy` with
`--no-fail-on-empty-changeset`, so repeating a command is harmless. `test.sh` lints the same
templates on every push.

A service deploy builds `backend/<service>` into an arm64 image, pushes it to the `sum-repository`
ECR repository as `<service>-arm64-<yyyymmdd-HHMM>` and `<service>-arm64-latest`, and deploys the
timestamped tag; `--skip-docker` deploys the `-latest` image already in ECR. A service folder
without its own `layers/` gets a copy of the Lambda layers, `backend/summaries-service/layers`,
for the build; the usage service reads Postgres and Redis through the `dbData` layer.

## After a new host: the Kafka broker address

The broker and the databases are addressed by the host's public DNS name. At first boot the host
reads its own name from the instance metadata service and writes it into Redpanda's advertised
addresses and the statistics loop, so the host side follows a replaced instance by itself. The
Lambdas receive the name as `ECS_HOST_ADDRESS` in `scripts/lib/stack-params.sh`, the one pinned
copy, from which the broker and database parameters derive; the ECS tasks take the host's
private name from `scripts/deploy-ecs.sh`. A Kafka client connects to the bootstrap address and
is redirected to the advertised one, so a pinned name that no longer matches the host breaks
every producer and consumer.

The host has no Elastic IP, so each new instance gets a new name. After the cluster stack is up,
or after the Auto Scaling group has replaced the instance:

1. Read the name:
   `aws ec2 describe-instances --filters Name=tag:Name,Values=sum-ecs-c__db-server
   Name=instance-state-name,Values=running --query 'Reservations[].Instances[].PublicDnsName'
   --output text`
2. Set `ECS_HOST_ADDRESS` in `scripts/lib/stack-params.sh` to the name, and
   `ECS_HOST_PRIVATE_ADDRESS` in `scripts/deploy-ecs.sh` to the private one (`PrivateDnsName` in
   the same query).
3. `./scripts/deploy.sh` for the Lambdas, and a service deploy for the ECS tasks.

## Undeploying

Services first, then the cluster; each command waits for its stack deletion:

```bash
./scripts/deploy-ecs.sh --service usage-service --undeploy
./scripts/deploy-ecs.sh --undeploy
```

Deleting the cluster terminates the host and the Kafka topics on it. The Lambdas keep publishing
to the pinned broker name; only those deployed with `KAFKA_MAY_BE_DOWN` tolerate its absence.
`scripts/aws-helper.sh --list-resources` confirms that nothing was left behind.
