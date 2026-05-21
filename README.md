# Personal Blog Backend (AWS)

Serverless backend for a personal blog, provisioned with the AWS Cloud Development Kit (CDK) in TypeScript. The design separates public read traffic from administrative write operations, shares persistence logic through a Lambda Layer, and emits telemetry to a dedicated observability stack.

## Architecture Overview

```
User
  |
  v
Amazon API Gateway
  |
  +---> Blog Fetch (Lambda)  ----+
  |                              |
  +---> Blog Admin (Lambda)  ----+---> Lambda Layer (shared)
  |                              |
  +------------------------------+---> Amazon DynamoDB

Blog Fetch, Blog Admin, DynamoDB, and Lambda Layer
  |
  +---> Observability (CloudWatch, X-Ray, CloudWatch Logs Insights)
```

<img width="1536" height="1024" alt="07877863-bd2d-4826-95f2-d840acced99a" src="https://github.com/user-attachments/assets/a5f8822e-8180-4ccc-98cd-0b5c651e814b" />


## How It Works

### Request path

1. **Client** — A user or admin client sends HTTP requests to the API.
2. **Amazon API Gateway** — Terminates TLS, routes requests, and invokes the appropriate Lambda integration.
3. **Compute** — Two functions handle distinct responsibilities:
   - **Blog Fetch** — Read-only operations (e.g. list posts, get post by slug or ID).
   - **Blog Admin** — Mutations and management (e.g. create, update, delete posts).
4. **Lambda Layer (shared)** — Common code used by both functions: data access helpers, models, validation, and shared dependencies. This avoids duplication and keeps read and write handlers aligned on the same persistence contract.
5. **Amazon DynamoDB** — Single-table (or domain-scoped) storage for blog content and metadata. Both Lambdas and the layer’s repository logic read and write through the same access patterns.

### Observability

Operational data flows from the compute and data layers into a separate observability boundary:

| Service | Role |
|--------|------|
| **Amazon CloudWatch** | Logs, custom metrics, and alarms for errors, latency, and capacity signals. |
| **AWS X-Ray** | Distributed tracing and performance analysis across API Gateway, Lambda, and DynamoDB calls. |
| **Amazon CloudWatch Logs Insights** | Ad-hoc log analysis and queries (e.g. failed admin actions, slow reads). |

Lambda functions are configured to emit structured logs and traces so incidents can be correlated from the edge (API Gateway) through to DynamoDB.

## Design Principles

- **Separation of concerns** — Public reads (`Blog Fetch`) and administrative writes (`Blog Admin`) are isolated at the API and function level, which simplifies authorization, scaling profiles, and blast-radius control.
- **Shared domain logic** — The Lambda Layer centralizes DynamoDB access and business rules so both functions stay consistent without copying code.
- **Serverless operations** — No servers to patch; pay-per-use scaling aligned with blog traffic patterns.
- **Observable by default** — Metrics, logs, and traces are first-class so the system can be operated without SSH or host access.

## Repository Layout

| Path | Description |
|------|-------------|
| `bin/blog.ts` | CDK application entry point |
| `lib/blog-stack.ts` | Infrastructure stack (API Gateway, Lambdas, Layer, DynamoDB, observability wiring) |
| `docs/architecture.png` | Reference architecture diagram |
| `test/` | Unit tests (Jest) |

Lambda function source and layer assets are added under this project as the implementation progresses; the stack in `lib/blog-stack.ts` is the single place to define AWS resources.

## Prerequisites

- Node.js 22+ (LTS recommended)
- AWS CLI configured with credentials for the target account and region
- AWS CDK CLI (`npm install -g aws-cdk` or use `npx cdk`)

Bootstrap the CDK environment in your account/region once if you have not already:

```bash
npx cdk bootstrap
```

## Development

Install dependencies and compile TypeScript:

```bash
npm install
npm run build
```

Run tests:

```bash
npm run test
```

Watch mode for local TypeScript builds:

```bash
npm run watch
```

## Deployment

Synthesize the CloudFormation template:

```bash
npx cdk synth
```

Compare the deployed stack with the current app definition:

```bash
npx cdk diff
```

Deploy to the default AWS account and region:

```bash
npx cdk deploy
```

Destroy the stack when tearing down the environment (use with care in shared accounts):

```bash
npx cdk destroy
```

## Security Notes

- Restrict **Blog Admin** routes at API Gateway (e.g. API keys, JWT authorizer, or IAM) before production use.
- Apply least-privilege IAM roles per Lambda; the shared layer should not broaden permissions beyond what each function needs.
- Enable encryption at rest for DynamoDB and consider AWS WAF on API Gateway if the API is public on the internet.

