# events-streaming-function — Module Instructions

## Purpose

Lambda function that exposes a REST API (`POST /api/events`) and publishes click events to a Kinesis Data Stream. Also contains a stub generator that creates random events for load testing.

## File Map

```
events-streaming-function/
├── src/
│   ├── main.py              # Lambda entry point — routes HTTP vs direct invocations
│   ├── controller/
│   │   ├── app.py           # Flask app with POST /api/events route
│   │   └── stub.py          # Kinesis PutRecord logic + random event generator
├── terraform_scripts/
│   ├── lambda.tf            # Lambda function, IAM role, CloudWatch log group
│   ├── input.tf             # Variables: repo_name, environment, region, kinesis_ds_name/arn
│   ├── output.tf            # Outputs: lambda ARN, lambda name
│   └── main_provider.tf     # AWS provider block
└── Dockerfile               # Lambda container image (python:3.9)
```

## Event Schema

Events published to Kinesis:
```json
{
  "user_id": "string",
  "page": "string",
  "timestamp": "ISO-8601 string"
}
```

## IAM Permissions (least-privilege)

This Lambda only needs `kinesis:PutRecord` on the target stream. Do not add broader Kinesis permissions.

## Local Testing

```bash
# Install deps
pip install -r requirements.txt

# Invoke the Flask route directly (no Lambda wrapper)
python -c "
from src.controller.app import app
with app.test_client() as c:
    r = c.post('/api/events', json={'user_id': 'u1', 'page': 'home'})
    print(r.status_code, r.json)
"
```

## Terraform Inputs Required

```hcl
repo_name        = "your-ecr-repo-name"
environment      = "dev"
region           = "us-east-1"
kinesis_ds_name  = "<from resources module output>"
kinesis_ds_arn   = "<from resources module output>"
```

## Common Pitfalls

- The `kinesis_ds_arn` must be passed explicitly — the Lambda IAM policy uses it to scope `kinesis:PutRecord`
- Flask is initialized at module level in `app.py` so it persists across Lambda warm invocations
- `aws-wsgi` translates API Gateway proxy events to WSGI — don't bypass it with raw event parsing
