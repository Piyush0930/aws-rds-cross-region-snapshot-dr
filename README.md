# AWS RDS Cross-Region Snapshot DR POC

A proof of concept that automatically creates RDS PostgreSQL snapshots in Mumbai and copies them to Hyderabad every 4 minutes, using AWS Lambda and EventBridge Scheduler.

| | |
|---|---|
| Primary region | Mumbai (`ap-south-1`) |
| Secondary region | Hyderabad (`ap-south-2`) |
| Database | `rds-snapshot-poc-pg` (RDS PostgreSQL) |

## Objective

Show how database backups can be created automatically in a primary region and copied to a secondary region for disaster recovery (DR).

## Workflow

1. EventBridge Scheduler triggers the Lambda function every 4 minutes.
2. Lambda checks the existing manual snapshots in Mumbai.
3. If a snapshot is available, Lambda checks whether its copy already exists in Hyderabad.
4. If no copy exists, Lambda starts a cross-region copy.
5. Lambda creates a new manual snapshot in Mumbai.
6. On the next run, Lambda checks for other eligible snapshots.

## AWS Resources

### RDS database

| Setting | Value |
|---|---|
| DB identifier | `rds-snapshot-poc-pg` |
| Engine | PostgreSQL 18.3 |
| Instance class | `db.m7g.large` |
| Storage | 20 GiB gp3 |
| Availability | Single AZ |
| Region | Mumbai (`ap-south-1`) |

### KMS keys

Snapshots are encrypted. When copying to another region, the Hyderabad KMS key is specified so the copy is encrypted with the destination region's key.

| Region | Key |
|---|---|
| Mumbai | `aws/rds` |
| Hyderabad | `aws/rds` |

### IAM role

Role: `RDS-Snapshot-POC-Lambda-Role-Copy`

| Policy | Purpose |
|---|---|
| `AWSLambdaBasicExecutionRole` | AWS managed policy for Lambda logging |
| `RDS-Snapshot-POC-Policy` | Create, describe, copy and delete RDS snapshots; describe RDS instances; add and list resource tags |
| `RDS-Snapshot-POC-KMS-Policy` | `kms:DescribeKey`, `kms:CreateGrant`, `kms:Decrypt`, `kms:Encrypt`, `kms:GenerateDataKey`, `kms:ReEncrypt*` for both regional KMS keys |

![IAM role](docs/images/01-iam-role.png)

## Lambda Function

| Setting | Value |
|---|---|
| Function name | `rds-snapshot-poc-lambda` |
| Runtime | Python 3.15 |
| Region | Mumbai (`ap-south-1`) |
| Architecture | x86_64 |
| Timeout | 60 seconds |
| Execution role | `RDS-Snapshot-POC-Lambda-Role-Copy` |
| Reserved concurrency | 1 |

![Lambda function](docs/images/02-lambda-function.png)

### Code overview

- **`create_mumbai_snapshot()`** checks that the RDS instance is available, generates a timestamp-based name, creates a manual snapshot in Mumbai and adds project and environment tags. Example: `poc-rds-snapshot-poc-pg-20261118-063139`.
- **`copy_available_snapshots()`** lists manual snapshots in Mumbai with the POC prefix, checks that each is `available`, checks whether the copy already exists in Hyderabad, and if not, starts the cross-region copy using the Hyderabad KMS key. Example: `poc-rds-snapshot-poc-pg-20261118-053209` is copied to `hyd-poc-rds-snapshot-poc-pg-20261118-053209`.
- **`lambda_handler()`** is the entry point. It copies eligible snapshots first, then creates a new Mumbai snapshot, and returns the results.

## Manual Testing

The cross-region copy was tested manually before automation.

| Parameter | Result |
|---|---|
| Source region | Mumbai |
| Destination region | Hyderabad |
| Source snapshot | `poc-rds-snapshot-poc-pg-20261118-052926` |
| Destination snapshot | `new-hydra` |
| Destination status | Available |
| Snapshot size | 892.5 MiB |

Example Lambda response:

```json
{
  "copy_started": [],
  "already_copied": [],
  "copy_skipped": [],
  "copy_errors": [],
  "new_mumbai_snapshot": {
    "status": "creating",
    "snapshot": "poc-rds-snapshot-poc-pg-20261118-063139"
  }
}
```

`copy_started` was empty because no eligible source snapshots were available at that time. A new snapshot shows `creating` first because snapshot creation is asynchronous. A snapshot can only be copied once its status is `available`.

## EventBridge Scheduler

| Setting | Value |
|---|---|
| Schedule name | `rds-snapshot-poc-every-4-min` |
| Schedule type | Recurring |
| Schedule expression | `rate(4 minutes)` |
| Time zone | Asia/Calcutta |
| Flexible time window | Off |
| Target | AWS Lambda: `rds-snapshot-poc-lambda` |
| Payload | `{}` |
| Schedule state | Enabled |
| Action after completion | NONE |
| Execution role | Automatically generated Scheduler role |
| Dead-letter queue | None |
| Encryption | `aws/scheduler` |

![EventBridge schedule](docs/images/03-eventbridge-schedule.png)

## Results

The schedule started at 12:30 (UTC+05:30) and a new snapshot was created every 4 minutes.

**Mumbai (`ap-south-1`):** five manual snapshots between 12:30 and 12:46. Four were Available; the latest was still Creating.

![Mumbai snapshots](docs/images/04-mumbai-snapshots.png)

**Hyderabad (`ap-south-2`):** the four available snapshots were copied with the prefix `hyd-` and all show Available. The snapshot still being created in Mumbai is copied on a later run.

![Hyderabad snapshots](docs/images/05-hyderabad-snapshots.png)

## Important Notes

- **Retention:** the code does not delete old snapshots. A 4-minute schedule creates about 360 snapshots per day, which can increase storage costs and reach snapshot quota limits. Add a retention strategy before running continuously.
- **Lambda does not wait:** it starts the snapshot and exits. The next run checks whether the previous snapshot is available.
- **Copying is asynchronous:** a copy may show `creating` for a while. This does not mean it failed.
- **Scope:** this POC covers snapshot creation and cross-region copy only. Automatic restore, application failover, DNS switching and end-to-end DR testing are not implemented.
- **Schedule:** the 4-minute interval is for testing. It does not mean AWS guarantees a 4-minute RPO, or that every snapshot is fully copied within 4 minutes.

## Status

| Component | Status |
|---|---|
| RDS PostgreSQL (Mumbai) | Created |
| KMS keys (Mumbai and Hyderabad) | Configured |
| IAM role and policies | Configured |
| Lambda function and Python code | Deployed and tested |
| Manual cross-region snapshot copy | Tested successfully |
| EventBridge Scheduler | Created and enabled |
| Automated snapshot creation and cross-region copy | Working |
| Retention and cleanup | Pending |
| Snapshot storage cost review | Pending |

## Repository Structure

```
.
├── README.md
├── lambda_function.py
└── docs/
    └── images/
        ├── 01-iam-role.png
        ├── 02-lambda-function.png
        ├── 03-eventbridge-schedule.png
        ├── 04-mumbai-snapshots.png
        └── 05-hyderabad-snapshots.png
```
