# Development & deployment

This document details the local setup, the tests and the operations scripts behind the
[README](../README.md)'s Getting started and Scripts sections; [Architecture](architecture.md)
describes what the scripts deploy. Each script's header comment is its full documentation, and
`--help` prints it.

**Contents:** [Working copy location](#working-copy-location) · [Frontend](#frontend) ·
[Configuration](#configuration) · [Backend tests](#backend-tests) ·
[Deployment scripts (scripts/)](#deployment-scripts-scripts) ·
[What must exist outside the stack](#what-must-exist-outside-the-stack) · [Logs](#logs)

## Working copy location

Keep the checkout on the WSL ext4 filesystem (`~/projects/serverless/Summaries.AI`) rather than
under `/mnt/c`. WSL reaches Windows drives through a file-sharing protocol that makes every file
open a round trip to the Windows side, and `npm ci`, the TypeScript compile and the test run open
thousands of files.

`node_modules` folders record absolute paths, so each checkout installs its own instead of
receiving a copy.

## Frontend

```bash
./scripts/lib/app-config.sh     # on a fresh checkout: point appConfig.json at the branch's stack
cd frontend && npm install && npm start
```

The dev server listens on `http://localhost:3000`, a callback the stack's Cognito app client
allows, and calls the deployed backend named in `frontend/src/appConfig.json`, which
`lib/app-config.sh` writes from the stack's outputs and every deploy rewrites.
`frontend/src/appConfig.example.json` shows the file's shape. A bare `npm run build` deploys
nothing; the production frontend ships only through `deploy.sh --frontend`.

## Configuration

`./scripts/init-env.sh` creates the gitignored settings files under `scripts/lib/` from their
`.example.sh` files; each setting's comment says what it holds and whether it is required.

- `aws-config.sh` — the access key pair and region of the target account.
- `secrets.sh` — the PostgreSQL and Redis passwords the stack is deployed with, and the
  third-party keys of the scripts that call those services themselves.
- `params.sh` — the addresses: the mail sender and the RAG query tenant.

Every other setting is committed in `scripts/lib/stack-params.sh`, which lists every parameter
the stack is deployed with: `sam deploy` keeps a stack's previous value for any parameter the
overrides leave out. `AppSecretValue` is left out on purpose, since passing it empty would make
the main stack delete the secret. The app's name (`sum`), its region and the log retention
(14 days) are in `scripts/lib/app-env.sh`. An edit to any of them takes a `deploy.sh` run.

A bare `aws` command in your own shell has no credentials until it sources
`scripts/lib/aws-config.sh`; the scripts source it for you. A script that loads a settings file
first stops if git would track the file or a tracked file holds one of its secrets or
addresses. It also sets `core.hooksPath` to `.githooks`, whose `pre-commit` hook makes the same
check on every commit's staged changes.

## Backend tests

`scripts/test.sh` is the whole quality gate, and the GitHub workflow
`.github/workflows/ci.yml` only installs the tools and runs it, on pushes to `main` and on pull
requests. It runs a gitleaks scan of the files git records, `bash -n` on every script and the
pre-commit hook, shellcheck from a pinned image, `sam validate --lint` on every template,
`npm ci` where lockfiles changed, the transpile with its hard-link check, the backend suites in
`backend/tests/` with the Node test runner, the frontend tests (`react-scripts test`) and the
frontend build. It needs no AWS access; when `appConfig.json` is absent, the example file stands
in for the run.

## Deployment scripts (`scripts/`)

Install the AWS CLI v2, the AWS SAM CLI, Node.js with npm, `jq` and git, each on `PATH`;
`deploy.sh` stops naming the first one missing. Docker is needed by `test.sh` (gitleaks,
shellcheck) and by `deploy-ecs.sh` (the image build).

No script takes an environment argument. The git branch selects the stack every script acts on:
`main` targets `sum` (stage `prod`), any other branch `sum-fb` (stage `dev`), which reuses the
main stack's Summaries table, summaries bucket and its distribution; an exported `STACK_NAME`
wins.

- `deploy.sh` — validates and lints `scripts/template.yaml`, runs
  `npm ci` in every Lambda and layer folder whose `package-lock.json` changed, compiles the
  TypeScript through `lib/transpile.sh`, then runs `sam build`
  and `sam deploy`. `--frontend` also builds the React app, syncs it to the frontend bucket and
  invalidates the CloudFront distribution; `--frontend-only` does only that; `--sync` runs
  `sam sync` and is refused on `main`. Every run rewrites `frontend/src/appConfig.json` through
  `lib/app-config.sh`.
- `deploy-ecs.sh` — deploys or deletes the ECS cluster stack and
  the usage-service stack, building and pushing the service image to ECR. Both deploy once and
  stay up across backend deploys ([ECS deployment](ecs-deployment.md)).
- `aws-helper.sh` — AWS operations on the branch's stack, the same
  file in every AWS repo: function logs, log retention, `--delete-stack [--empty]`,
  `--list-resources`, `--xray on|off` and `--prune-layers`. This repo's own options are in
  `lib/aws-helper-app.sh`: `--lambda-concurrency` (the
  concurrency, invocation and error metrics of one function) and `--enable-bedrock-models` (the
  Marketplace agreement, quota and access of each Claude model the catalog offers). `--help`
  prints both headers.
- `test.sh` — the quality gate, above.
- `tools/` — hands-on tools: `sql-helper.sh` (psql against the
  app database), `rag-maintenance.sh` (reconciles the RAG documents in S3 with their chunks in
  Pinecone and smoke-tests the RAG query API) and `publish-docs.sh` (publishes the README and
  `docs/` to the `Summaries.AI-public` repository).

`scripts/lib/app-env.sh` holds what the scripts share: the `--help` printer, the settings-file
loaders, which refuse a file git tracks, and the branch's stack selection.
`lib/transpile.sh` recreates the hard links of the shared type files, then compiles each
backend package's TypeScript with its own `tsconfig.json`; the compiled `.js` files are
gitignored.

## What must exist outside the stack

A deploy depends on parts no script of this repo creates:

- The shared stacks `vsdb-cognito` (the user pool with Google sign-in and its domain) and
  `vsdb-lambda-layers` (the AWS SDK v3 and CORS headers layers). Their identifiers, the layer
  versions included, are in `stack-params.sh`; [Sign-in](sign-in.md#setting-up-google) covers
  the pool in a new account.
- PostgreSQL and Redis on the ECS host, run by the vpc-subnets-sgs repository, and the host's
  public DNS name in `ECS_HOST_ADDRESS` of `stack-params.sh`
  ([ECS deployment](ecs-deployment.md)).
- The Pinecone indexes `summaries-index` and `documents-index`, 1,536 dimensions, cosine.
- The Secrets Manager secret `/sum/app-secrets`, holding `OPENAI_API_KEY`, `PINECONE_API_KEY`,
  `OPENWEATHER_API_KEY`, `STRIPE_PUBLISHABLE_KEY`, `STRIPE_SECRET_KEY` and
  `STRIPE_WEBHOOK_SECRET`. The deployed functions read their keys from it, and
  `lib/app-config.sh` reads the publishable key. The main stack owns it; no script writes it,
  so a new account needs it created first.
- `SES_SENDER` of `params.sh` as an identity SES has verified, since every mail goes out from
  it.
- The Stripe webhook, registered once in the Stripe dashboard ([Billing](billing.md#routes)).

## Logs

Each function writes to `/aws/lambda/<stack>/<logical id>`, a log group the template declares
and keeps for 14 days, so a deploy leaves the logs alone; `aws-helper.sh --delete-stack`
deletes them with the stack.
`./scripts/aws-helper.sh --logs <function> --since 1d --errors` shows one function's warnings
and errors; `--trim-logs` deletes the stack's log streams on request.
