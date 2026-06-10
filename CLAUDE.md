# Realtime Streaming Analytics — Claude Code Instructions

## Project Overview

Real-time clickstream event pipeline on AWS. Users hit a REST API, events are published to Kinesis Data Streams, a second Lambda consumes and writes to Amazon Timestream, and Kinesis Firehose archives raw events to S3.

**Data flow:**
```
HTTP POST /api/events
  → events-streaming-function (Lambda + Flask)
  → Kinesis Data Stream (1 shard, 24-hr retention)
  → events-preprocessing-function (Lambda trigger)
  → Amazon Timestream (live analytics)
  → Kinesis Firehose → S3 (GZIP archive)
```

## Repository Layout

```
.
├── CLAUDE.md                          # ← you are here
├── .mcp.json                          # MCP servers: aws-knowledge, terraform
├── events-streaming-function/         # Lambda 1: REST API + Kinesis producer
│   ├── src/
│   │   ├── main.py                    # Lambda handler
│   │   ├── controller/app.py          # Flask route POST /api/events
│   │   └── controller/stub.py        # Random event generator + Kinesis put
│   ├── terraform_scripts/             # IaC for this Lambda
│   └── Dockerfile
├── events-preprocessing-function/     # Lambda 2: Kinesis consumer + Timestream writer
│   ├── src/
│   │   ├── main.py                    # Lambda handler
│   │   ├── controller/etl.py          # Decode Kinesis records → Timestream schema
│   │   └── infra/timestreamdb.py      # TimestreamDB write client
│   ├── terraform_scripts/             # IaC for this Lambda
│   └── Dockerfile
└── resources/
    └── terraform_scripts/             # Shared infra: Kinesis DS, Firehose, S3, CloudWatch
```

## MCP Servers Available

This project has two MCP servers configured in `.mcp.json`:

| Server | Purpose |
|---|---|
| `aws-knowledge-mcp-server` | Query AWS docs, service limits, regional availability |
| `terraform` | Look up Terraform provider/module versions, policy details |

Use these when you need to verify AWS service behavior, check Timestream write limits, look up Kinesis shard capacity, or find the latest Terraform AWS provider version.

## Tech Stack

- **Runtime**: Python 3.9 (AWS Lambda container image)
- **Web framework**: Flask 2.3.2 (via aws-wsgi adapter)
- **AWS services**: Lambda, Kinesis Data Streams, Kinesis Firehose, Timestream, S3, CloudWatch, IAM
- **IaC**: Terraform (AWS provider)
- **SDK**: boto3

## Common Commands

### Terraform (run inside each `terraform_scripts/` directory)
```bash
terraform init
terraform validate
terraform plan -var-file=vars.tfvars
terraform apply -var-file=vars.tfvars
terraform destroy -var-file=vars.tfvars
```

### Docker (build Lambda images locally)
```bash
docker build -t events-streaming-function ./events-streaming-function
docker build -t events-preprocessing-function ./events-preprocessing-function
```

### Python (local testing)
```bash
pip install -r events-streaming-function/requirements.txt
pip install -r events-preprocessing-function/requirements.txt
```

### Test the streaming API (after deploy)
```bash
curl -X POST https://<api-url>/api/events \
  -H "Content-Type: application/json" \
  -d '{"user_id": "u123", "page": "home"}'
```

## Deployment Order

Infrastructure must be provisioned in this order to avoid dependency errors:

1. `resources/terraform_scripts/` — Kinesis DS, Firehose, S3 bucket
2. `events-streaming-function/terraform_scripts/` — Producer Lambda (needs Kinesis DS ARN)
3. `events-preprocessing-function/terraform_scripts/` — Consumer Lambda (needs Kinesis DS ARN + Timestream)

## Coding Conventions

- Lambda handlers always have the signature `handler(event, context)` in `main.py`
- Kinesis records are base64-encoded JSON; always decode with `base64.b64decode` before `json.loads`
- Timestream writes use `MeasureValueType: DOUBLE` for numeric measures; dimensions are strings
- Flask routes live in `controller/app.py`; business logic in `controller/stub.py` or `controller/etl.py`
- Terraform variable names follow `snake_case`; resource names use `var.environment` prefix for multi-env support
- IAM policies follow least-privilege: streaming Lambda only has `kinesis:PutRecord`, preprocessing Lambda has `kinesis:GetRecords` + Timestream write

## Key Variables (Terraform)

| Variable | Used in | Purpose |
|---|---|---|
| `repo_name` | both lambdas | ECR image tag / naming |
| `environment` | all modules | `dev`/`staging`/`prod` prefix |
| `region` | all modules | AWS region |
| `kinesis_ds_name` | streaming | Kinesis stream name |
| `kinesis_ds_arn` | streaming | Kinesis stream ARN |
| `db_name` | preprocessing | Timestream database name |
| `table_name` | preprocessing | Timestream table name |

## Things to Watch Out For

- Kinesis base64 decoding: records arrive as `event["Records"][i]["kinesis"]["data"]`
- Timestream requires dimensions as `[{"Name": ..., "Value": ...}]` — not a plain dict
- The Firehose → S3 path uses GZIP; don't forget `-d` flag when reading archives locally
- Lambda cold starts: Flask app is initialized outside the handler to reuse across invocations
- IAM propagation delay: after `terraform apply`, wait ~10 s before invoking Lambdas

## Sub-agents Available

See `.claude/agents/` for specialized sub-agents:
- `terraform-deployer` — plan/apply Terraform modules with dependency ordering
- `lambda-developer` — develop and test Lambda functions locally
- `infra-reviewer` — review AWS IaC for security, cost, and best practices
