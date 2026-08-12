[README.md](https://github.com/user-attachments/files/30991717/README.md)[Uploadin# Lakehouse E-commerce Data Pipeline

> An event-driven AWS data lakehouse pipeline that ingests CSV/Excel e-commerce data, validates and transforms it with AWS Glue/PySpark, stores curated datasets in Delta Lake on Amazon S3, and orchestrates processing with AWS Step Functions.

## Overview

This project demonstrates an end-to-end cloud data engineering workflow for **orders, products, and order-items** data.

The pipeline is designed around:

- Event-driven ingestion from Amazon S3
- CSV and multi-sheet Excel processing
- Automated validation and rejected-record handling
- PySpark-based ETL with AWS Glue
- Delta Lake storage with MERGE/upsert operations
- Partitioned data storage
- AWS Step Functions orchestration
- Glue Data Catalog and Athena querying
- CloudWatch logging/metrics and SNS notifications
- Automated testing with pytest
- GitHub Actions CI/CD

### What the pipeline solves

Raw e-commerce files often arrive in different formats and may contain invalid, duplicate, or inconsistent records. This pipeline automatically moves data through ingestion, conversion, validation, transformation, deduplication, curated storage, cataloging, and downstream validation.

---

## Architecture

![Architecture Diagram](images/architecture_diagram.svg)

### End-to-end flow

```text
CSV / Excel Files
       |
       v
Amazon S3 - Incoming
       |
       v
S3 Event
       |
       v
AWS Lambda
   |       |
   |       +--> CSV -> Processing
   |
   +--> Excel -> Pandas/OpenPyXL -> CSV -> Processing
       |
       v
AWS Step Functions
       |
       +-------------------------------+
       |               |               |
       v               v               v
 Orders ETL      Products ETL    Order Items ETL
       |               |               |
       +---------------+---------------+
                       |
                       v
              AWS Glue + PySpark
                       |
          +------------+------------+
          |                         |
          v                         v
   Validation /              Deduplication /
   Rejection handling        Transformation
          |                         |
          +------------+------------+
                       |
                       v
             Delta Lake on S3
                       |
              +--------+--------+
              |                 |
              v                 v
       Glue Data Catalog     Athena
              |
              v
       Analytics / Queries

CloudWatch Logs & Metrics
          |
          v
      SNS Alerts

GitHub Actions
      |
      +--> Tests
      +--> Spark syntax validation
      +--> AWS deployment steps
```

---

## Key Features

### 1. Multi-format ingestion

The ingestion layer accepts:

- `.csv`
- `.xlsx`
- `.xls`

Excel workbooks can contain multiple sheets. The Lambda processing layer reads the workbook and converts non-empty sheets into CSV files before downstream ETL.

### 2. Event-driven processing

An S3 upload to the `incoming/` path triggers the Lambda workflow.

The Lambda function:

1. Reads the S3 event.
2. Validates the file location and extension.
3. Identifies the dataset type from the file path/name.
4. Converts Excel sheets to CSV where required.
5. Places files in the processing area.
6. Starts an AWS Step Functions execution.

### 3. AWS Glue + PySpark ETL

Three ETL jobs process the main datasets:

```text
src/glue_jobs/
├── orders_etl.py
├── products_etl.py
└── order_items_etl.py
```

The ETL layer performs:

- Schema handling
- Data-type validation
- Null/data-quality checks
- Business-rule validation
- Rejected-record logging
- Deduplication
- Data transformation
- Delta Lake writes
- MERGE/upsert processing

### 4. Deduplication

The orders pipeline uses a window function over `order_id` and orders records by `order_timestamp` to retain the latest record.

This provides deterministic handling of duplicate order records.

### 5. Delta Lake

Processed datasets are stored as Delta tables on Amazon S3.

The implementation supports:

- ACID-style transactional storage through Delta Lake
- MERGE/upsert operations
- Schema alignment
- Partitioned initial writes
- Incremental updates to existing Delta tables

Orders and order-items are partitioned by:

```text
year / month
```

Products are partitioned by:

```text
department
```

### 6. Workflow orchestration

AWS Step Functions coordinates the pipeline stages and handles:

- File validation
- ETL job execution
- Parallel dataset processing
- Retries
- Failure branches
- Glue crawler execution
- Athena validation
- Success/failure notifications

### 7. Data catalog and querying

AWS Glue Data Catalog is used to discover and maintain metadata for processed datasets.

Amazon Athena provides SQL-based querying and validation over the curated data.

Example:

```sql
SELECT
    order_value_category,
    COUNT(*) AS order_count,
    AVG(total_amount) AS avg_order_value
FROM lakehouse_ecommerce_db.orders
GROUP BY order_value_category;
```

### 8. Monitoring and alerting

The project integrates AWS CloudWatch for logs/metrics and SNS for pipeline notifications.

This provides visibility into:

- Lambda processing
- Glue ETL execution
- Validation failures
- Pipeline errors
- Workflow status

### 9. Automated testing and CI/CD

The repository contains pytest-based tests for Lambda and Glue/PySpark components.

GitHub Actions runs automated checks for pushes and pull requests, including:

- Lambda unit tests
- PySpark/Glue unit tests
- Integration-test commands configured in CI
- Spark/Python syntax validation
- Coverage reporting
- AWS deployment steps on the `main` branch

---

## Data Model

The pipeline processes three primary datasets.

### Orders

Expected fields include:

```text
order_num
order_id
user_id
order_timestamp
total_amount
date
```

### Products

Expected fields include:

```text
product_id
department_id
department
product_name
```

### Order Items

Expected fields include:

```text
id
order_id
user_id
days_since_prior_order
product_id
add_to_cart_order
reordered
order_timestamp
date
```

---

## Project Structure

```text
Lakehouse-E-commerce-Data-Pipeline/
│
├── .github/
│   └── workflows/
│       └── deploy-pipeline.yaml
│
├── config/
│   └── aws_config.py
│
├── images/
│   ├── architecture_diagram.svg
│   └── step_functions.png
│
├── src/
│   ├── glue_jobs/
│   │   ├── orders_etl.py
│   │   ├── products_etl.py
│   │   └── order_items_etl.py
│   │
│   ├── lambda/
│   │   ├── s3_event_trigger.py
│   │   ├── file_processor.py
│   │   └── file_archiver.py
│   │
│   └── step_functions/
│       └── etl_orchestrator.json
│
├── tests/
│   ├── conftest.py
│   ├── test_basic.py
│   └── unit/
│       ├── test_glue_etl.py
│       ├── test_lambda_function.py
│       └── test_orders_validation.py
│
├── requirements.txt
├── requirements-lambda.txt
├── pytest.ini
└── README.md
```

---

## Technology Stack

| Area | Technologies |
|---|---|
| Cloud | AWS |
| Object Storage | Amazon S3 |
| ETL | AWS Glue, PySpark |
| Lakehouse Storage | Delta Lake |
| Serverless Processing | AWS Lambda |
| Orchestration | AWS Step Functions |
| Metadata | AWS Glue Data Catalog |
| SQL Analytics | Amazon Athena |
| Monitoring | Amazon CloudWatch |
| Alerting | Amazon SNS |
| Programming | Python, SQL |
| Data Processing | Pandas, PySpark |
| Testing | pytest, moto, pytest-cov |
| CI/CD | GitHub Actions |
| AWS SDK | boto3 |
| Security | AWS IAM |

---

## Configuration

The project uses a central AWS configuration class in:

```text
config/aws_config.py
```

Default development configuration includes:

```text
AWS_REGION=us-east-1
ENVIRONMENT=dev
RAW_BUCKET=lakehouse-raw-dev
PROCESSED_BUCKET=lakehouse-processed-dev
GLUE_DATABASE=lakehouse_ecommerce_db
STEP_FUNCTION_NAME=lakehouse-etl-orchestrator
```

For your own AWS account, update bucket names, IAM roles, Step Functions ARNs, Glue configuration, and other environment-specific values before deployment.

---

## Local Setup

### Prerequisites

- Python 3.9+
- AWS CLI
- An AWS account for cloud deployment
- Git

### Clone

```bash
git clone <your-repository-url>
cd Lakehouse-E-commerce-Data-Pipeline
```

### Create a virtual environment

```bash
python -m venv venv
```

Activate it on Linux/macOS:

```bash
source venv/bin/activate
```

Windows:

```bash
venv\Scripts\activate
```

### Install dependencies

```bash
pip install -r requirements.txt
```

---

## Testing

Run the complete configured test suite:

```bash
pytest tests/ -v
```

Run unit tests:

```bash
pytest tests/unit/ -v
```

Run a specific test module:

```bash
pytest tests/unit/test_glue_etl.py -v
```

Generate a coverage report:

```bash
pytest --cov=src tests/ --cov-report=html
```

The CI workflow also validates Python/Spark job syntax using:

```bash
python -m py_compile src/glue_jobs/*.py
```

---

## AWS Data Flow

### 1. Upload

Upload a supported file to an S3 incoming path:

```bash
aws s3 cp orders_data.xlsx \
s3://lakehouse-raw-dev/incoming/orders/
```

### 2. Event trigger

S3 invokes the Lambda event-processing function.

### 3. File conversion

For Excel files:

```text
Excel workbook
      |
      +--> Sheet 1 -> CSV
      +--> Sheet 2 -> CSV
      +--> Sheet N -> CSV
```

The resulting CSV files are placed in the processing area.

### 4. Orchestration

Lambda starts the Step Functions state machine with information about:

- Source bucket
- Original object key
- Converted files
- Dataset type
- Event timestamp

### 5. ETL

Step Functions starts the corresponding AWS Glue ETL job.

```text
Orders       -> orders_etl.py
Products     -> products_etl.py
Order Items  -> order_items_etl.py
```

### 6. Validation

Invalid records are separated and logged with rejection information.

### 7. Transformation

Valid records are transformed into curated structures suitable for analytics.

### 8. Delta Lake

Curated datasets are written to S3 as Delta tables.

### 9. Catalog

The Step Functions workflow starts the Glue crawler and waits for catalog completion.

### 10. Athena validation

Athena queries are used to validate the processed datasets.

### 11. Monitoring

CloudWatch records execution information and SNS can notify users of pipeline outcomes.

---

## Example Analytics Queries

### Average order value

```sql
SELECT
    AVG(total_amount) AS avg_order_value
FROM lakehouse_ecommerce_db.orders;
```

### Orders by category

```sql
SELECT
    order_value_category,
    COUNT(*) AS order_count
FROM lakehouse_ecommerce_db.orders
GROUP BY order_value_category;
```

### Products by department

```sql
SELECT
    department,
    COUNT(*) AS product_count
FROM lakehouse_ecommerce_db.products
GROUP BY department;
```

### Orders and order items

```sql
SELECT
    o.order_id,
    o.total_amount,
    COUNT(oi.id) AS item_count
FROM lakehouse_ecommerce_db.orders o
JOIN lakehouse_ecommerce_db.order_items oi
    ON o.order_id = oi.order_id
GROUP BY
    o.order_id,
    o.total_amount;
```

---

## CI/CD

The GitHub Actions workflow is located at:

```text
.github/workflows/deploy-pipeline.yaml
```

The workflow is triggered for:

```text
push -> main
pull_request -> main
```

The CI/CD process includes:

1. Checkout repository
2. Set up Python 3.9
3. Install dependencies
4. Run Lambda tests
5. Run PySpark/Glue tests
6. Run configured integration-test command
7. Validate Glue/PySpark Python syntax
8. Configure AWS credentials for deployment
9. Upload Glue ETL scripts
10. Package Lambda functions
11. Upload Step Functions definition

### Required GitHub secrets

For AWS deployment, configure the required repository secrets such as:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

Use appropriately scoped IAM credentials and follow your organization's security practices.

---

## Security Considerations

The project is structured to use AWS IAM roles and permissions for cloud resources.

For an actual deployment:

- Use least-privilege IAM policies.
- Do not commit AWS credentials.
- Store secrets in GitHub Secrets or AWS Secrets Manager.
- Enable S3 encryption.
- Restrict bucket access.
- Use separate development/staging/production resources where appropriate.
- Avoid hard-coding sensitive account-specific values.

---

## Error Handling

The pipeline includes error-handling paths for:

- Invalid input files
- Unsupported file formats
- Conversion failures
- Validation failures
- ETL failures
- Duplicate Step Functions executions
- Glue crawler failures
- Downstream validation failures

Step Functions uses retry and failure branches to prevent transient failures from immediately terminating the workflow.

---

## Limitations

Current implementation considerations include:

- Large Excel files may be constrained by Lambda execution limits because Excel conversion occurs in Lambda.
- The default configuration is development-oriented and uses `us-east-1`.
- Major schema changes may require updates to validation/transformation logic.
- AWS resources must be configured in the target account before the full cloud workflow can run.

---

## What I Learned

This project provided hands-on experience with:

- Designing event-driven data pipelines
- Building cloud ETL workflows with PySpark
- Working with AWS Glue and S3
- Implementing data validation and deduplication
- Using Delta Lake MERGE/upsert patterns
- Orchestrating multi-step workflows with Step Functions
- Managing metadata with Glue Data Catalog
- Querying curated data through Athena
- Adding monitoring and alerting
- Writing automated tests for data-processing components
- Automating deployment workflows with GitHub Actions

---

## Resume Highlight

**Cloud Lakehouse E-commerce Data Pipeline**

> Engineered an event-driven AWS lakehouse pipeline using **S3, Lambda, Step Functions and Glue/PySpark** to ingest CSV/Excel e-commerce data; implemented schema validation, deduplication, Delta Lake MERGE/upserts, partitioned storage, automated testing and monitoring for reliable downstream analytics.

### Core keywords

`AWS` `S3` `Glue` `PySpark` `Spark` `Delta Lake` `ETL` `Data Lakehouse` `Step Functions` `Lambda` `Athena` `Glue Data Catalog` `CloudWatch` `SNS` `Python` `SQL` `Data Quality` `Data Validation` `CI/CD` `GitHub Actions`

---

## Disclaimer

This repository is intended as a portfolio/learning implementation of a cloud data engineering architecture. AWS infrastructure, credentials, IAM policies, networking, monitoring and deployment configuration should be reviewed and hardened before use in a production environment.
g README.md…]()
