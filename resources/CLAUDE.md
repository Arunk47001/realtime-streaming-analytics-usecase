# resources — Shared Infrastructure Instructions

## Purpose

Provisions the shared AWS infrastructure that both Lambda functions depend on: Kinesis Data Stream, Kinesis Firehose → S3, and CloudWatch log groups. Must be deployed **before** either Lambda module.

## File Map

```
resources/
└── terraform_scripts/
    ├── kinesisDS.tf   # Kinesis Data Stream (1 shard, 24-hr retention) + Lambda trigger
    ├── kinesisFH.tf   # Kinesis Firehose (extended_s3 destination, GZIP compression)
    ├── cloudwatch.tf  # CloudWatch log group + stream for Firehose error logging
    ├── s3.tf          # S3 bucket for raw event archive
    ├── input.tf       # Variables: environment, source_bucket, lambda_function_name
    └── output.tf      # Outputs: s3_bucket, kinesis_ds_arn/name, kinesis_fh_arn/name
```

## Key Outputs (consumed by Lambda modules)

| Output | Used by |
|---|---|
| `kinesis_ds_name` | events-streaming-function (var `kinesis_ds_name`) |
| `kinesis_ds_arn` | events-streaming-function (var `kinesis_ds_arn`) |
| `kinesis_ds_arn` | events-preprocessing-function (Kinesis trigger) |
| `s3_bucket` | reference / monitoring |

## Terraform Inputs

```hcl
environment           = "dev"
source_bucket         = "my-analytics-archive-dev"
lambda_function_name  = "<preprocessing lambda name from that module's output>"
```

## Architecture Notes

- **Kinesis DS**: 1 shard = 1 MB/s ingest, 2 MB/s read. Scale shards if event volume exceeds this.
- **Kinesis Firehose**: buffers 60s or 5 MB before writing to S3. Files are GZIP-compressed.
- **S3 prefix**: Firehose writes to `s3://<bucket>/YYYY/MM/DD/HH/` by default.
- **Retention**: Kinesis DS retains data for 24 hours. Extend to 7 days (`retention_period = 168`) for replay capability.

## Common Pitfalls

- Deploy this module first — both Lambda modules reference outputs from here
- The Firehose IAM role needs `s3:PutObject` on the bucket and `kinesis:GetRecords` on the stream
- S3 bucket names must be globally unique — use `${var.environment}-${var.source_bucket}` pattern
- CloudWatch log group for Firehose errors is required even if you don't query it — Firehose will fail silently without it
