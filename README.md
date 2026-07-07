# 🚀 Lyft NYCTLC Data Pipeline

**End-to-end data pipeline for Lyft trip data from NYC Taxi & Limousine Commission (NYCTLC), deployed via GitHub Actions + AWS CloudFormation.**

Built as part of the [Data Engineering Hub](https://sachin.cloud) Silver Bootcamp.

---

## 📋 What This Pipeline Does

Pulls 6 months of Lyft rideshare trip data from NYCTLC's public dataset, transforms it through a medallion architecture (Raw → Curated → Aggregated), and makes it available for analyst queries via Amazon Athena.

### Data Flow
```
NYCTLC CloudFront (Source)
        ↓
Glue Python Shell (Ingestion) → S3 Raw Layer
        ↓
Glue Spark ETL (Transform) → S3 Curated Layer (Lyft only, filtered, enriched)
        ↓
Glue Spark ETL (Aggregate) → S3 Aggregated Layer (daily/hourly/route summaries)
        ↓
Athena (Query by Analysts)
```

### Orchestration
```
EventBridge (Monthly Schedule: 5th at 6 AM UTC)
        ↓
Step Functions (Full Pipeline: Ingest → Transform → Aggregate)

S3 Event (ingestion.done marker file)
        ↓
Lambda → Step Functions (Transform Pipeline: Transform → Aggregate)

On Success/Failure → SNS Email Alert
```

---

## 🏗️ Architecture

| Layer | Service | Purpose |
|-------|---------|---------|
| Ingestion | Glue Python Shell | Pull data from CloudFront, write to S3 raw |
| Transformation | Glue Spark ETL | Filter Lyft, DQ checks, derive columns, zone joins |
| Aggregation | Glue Spark ETL | Daily borough summary, hourly patterns, route summary |
| Catalog | Glue Crawlers + Data Catalog | Auto-register schemas for Athena |
| Query | Athena | Self-service analytics for data analysts |
| Orchestration | Step Functions | Sequential job execution with error handling |
| Scheduling | EventBridge Scheduler | Monthly cron trigger |
| Event-Driven | Lambda + S3 Events | Trigger transform on ingestion completion |
| Alerting | SNS | Email on success/failure |
| Observability | DynamoDB | Job audit records (records in/out, duration, status) |
| Cost Optimization | S3 Lifecycle | Raw→IA (30d), Raw→Glacier (90d), Rejected deleted (15d) |
| Deployment | CloudFormation + GitHub Actions | Infrastructure as Code + CI/CD |

---

## 📁 Repo Structure

```
├── .github/
│   └── workflows/
│       └── deploy.yml                  # GitHub Actions CI/CD workflow
├── cloudformation/
│   ├── infrastructure.yaml             # S3, IAM roles, DynamoDB, SNS
│   └── pipeline.yaml                   # Glue jobs, crawlers, Step Functions, Lambda, EventBridge
├── glue-scripts/
│   ├── backup-nyctlc-ingestion.py      # Ingestion: CloudFront → S3 raw (6 months)
│   ├── backup-raw-to-curated.py        # Transform: DQ filters, zone joins, derived columns
│   └── backup-curated-to-aggregated.py # Aggregate: 3 summary tables
├── lambda-scripts/
│   └── pipeline-trigger/
│       └── lambda_function.py          # S3 event → starts Transform Step Function
└── README.md
```

---

## 🚀 CI/CD Deployment

### How It Works

1. Developer pushes code to `development` branch
2. Creates Pull Request → merges to `main`
3. GitHub Actions triggers automatically on push to `main`
4. Deploys infrastructure stack (CloudFormation)
5. Uploads Glue + Lambda scripts to S3
6. Deploys pipeline stack (CloudFormation)
7. All resources ready — pipeline can be triggered

### GitHub Secrets Required

| Secret | Description |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | IAM user access key with admin permissions |
| `AWS_SECRET_ACCESS_KEY` | IAM user secret key |

### Deployment Steps (automated)

```
Checkout code → Configure AWS credentials → Deploy infrastructure.yaml
    → Upload scripts to S3 → Deploy pipeline.yaml → Done ✅
```

---

## 🗂️ AWS Resources Created

### Infrastructure Stack (`mission-deh-hof-infrastructure`)
- S3 bucket: `mission-deh-hof-nyctlc-{ACCOUNT_ID}` (with lifecycle rules)
- IAM roles: Glue, Step Functions, EventBridge Scheduler, Lambda
- DynamoDB: `mission-deh-hof-datalake-metadata`, `mission-deh-hof-job-audit`
- SNS topic: `mission-deh-hof-pipeline-alerts`

### Pipeline Stack (`mission-deh-hof-pipeline`)
- Glue databases: `nyctlc_raw`, `nyctlc_curated`, `nyctlc_aggregated`
- Glue jobs: ingestion (Python Shell), raw-to-curated (Spark), curated-to-aggregated (Spark)
- Glue crawlers: raw, curated, aggregated
- Step Functions: `mission-deh-hof-lyft-pipeline` (full), `mission-deh-hof-lyft-transform-pipeline` (event-driven)
- Lambda: `mission-deh-hof-pipeline-trigger` (S3 event → Step Function)
- EventBridge: `mission-deh-hof-pipeline-schedule` (monthly, disabled by default)

---

## 📊 Data Details

| Metric | Value |
|--------|-------|
| Source | NYCTLC HVFHV (High Volume For-Hire Vehicles) |
| Filter | Lyft only (`hvfhv_license_num = HV0005`) |
| Volume | ~35 million curated records (6 months) |
| Raw size | ~3 GB (6 parquet files) |
| Schedule | 5th of every month at 6 AM UTC |
| Event trigger | `raw/nyctlc/ingestion.done` marker file |

---

## 🧹 Teardown

Delete both CloudFormation stacks to remove all resources:
```bash
aws cloudformation delete-stack --stack-name mission-deh-hof-pipeline
aws cloudformation wait stack-delete-complete --stack-name mission-deh-hof-pipeline
aws cloudformation delete-stack --stack-name mission-deh-hof-infrastructure
aws cloudformation wait stack-delete-complete --stack-name mission-deh-hof-infrastructure
# Delete S3 bucket manually (must be empty first)
aws s3 rm s3://mission-deh-hof-nyctlc-{ACCOUNT_ID} --recursive
aws s3 rb s3://mission-deh-hof-nyctlc-{ACCOUNT_ID}
```

---

## 📜 License

© Sachin Chandrashekhar — [Data Engineering Hub](https://sachin.cloud). All Rights Reserved.

This code is provided as part of a paid training program. Unauthorized copying, distribution, or publication is strictly prohibited.

---

## 🏆 About

This project is part of the **DEH Silver Bootcamp** — a hands-on AWS Data Engineering program where students build production-grade pipelines from scratch.

Learn more: [sachin.cloud](https://sachin.cloud)
# Triggered deployment
