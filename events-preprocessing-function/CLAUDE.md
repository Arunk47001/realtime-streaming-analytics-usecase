# events-preprocessing-function — Module Instructions

## Purpose

Lambda function triggered by Kinesis Data Streams. Decodes incoming records, transforms them into Amazon Timestream schema, and writes page-view metrics. Acts as the analytics sink in the pipeline.

## File Map

```
events-preprocessing-function/
├── src/
│   ├── main.py              # Lambda entry point — iterates Kinesis Records batch
│   ├── controller/
│   │   └── etl.py           # Decode base64 Kinesis data → Timestream write records
│   └── infra/
│       └── timestreamdb.py  # Timestream boto3 client + write_records wrapper
├── terraform_scripts/
│   ├── lambda.tf            # Lambda, IAM role, Kinesis trigger, CloudWatch logs
│   ├── input.tf             # Variables: repo_name, environment, region, db_name, table_name
│   ├── output.tf            # Outputs: lambda ARN, lambda name
│   └── main_provider.tf     # AWS provider block
└── Dockerfile               # Lambda container image (python:3.9)
```

## Kinesis Record Decoding

Kinesis delivers records base64-encoded. Always decode before parsing:

```python
import base64, json

raw = event["Records"][i]["kinesis"]["data"]
payload = json.loads(base64.b64decode(raw).decode("utf-8"))
```

## Timestream Write Schema

```python
{
    "Dimensions": [
        {"Name": "user_id", "Value": payload["user_id"]},
        {"Name": "page",    "Value": payload["page"]},
    ],
    "MeasureName":      "page_view",
    "MeasureValue":     "1",
    "MeasureValueType": "DOUBLE",
    "Time":             str(int(time.time() * 1000)),
    "TimeUnit":         "MILLISECONDS",
}
```

## IAM Permissions

This Lambda needs:
- `kinesis:GetRecords`, `kinesis:GetShardIterator`, `kinesis:DescribeStream`, `kinesis:ListStreams` — Kinesis trigger
- `timestream:WriteRecords`, `timestream:DescribeEndpoints` — Timestream writes

## Terraform Inputs Required

```hcl
repo_name    = "your-ecr-repo-name"
environment  = "dev"
region       = "us-east-1"
db_name      = "<timestream database name>"
table_name   = "<timestream table name>"
```

## Common Pitfalls

- Timestream `DescribeEndpoints` must be in the IAM policy — the SDK calls it automatically before writes
- `MeasureValue` must be a string even for numeric types; cast with `str(value)`
- Kinesis trigger in `lambda.tf` uses `StartingPosition = "LATEST"` — change to `TRIM_HORIZON` if you need to reprocess historical records
- Batch failures: if one record in the batch fails to write, the entire batch is retried; make ETL idempotent
