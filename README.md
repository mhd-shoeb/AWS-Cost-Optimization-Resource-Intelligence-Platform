# AWS Cost Optimization & Resource Intelligence Platform

## Overview

An automated AWS serverless platform that monitors AWS costs, scans resources across enabled regions, identifies potential optimization opportunities, stores reports in Amazon S3, and provides SQL-based cost analysis through Amazon Athena.

## Architecture

<img width="1536" height="1024" alt="AWS Cost Optimization   Resource Intelligence Platform" src="https://github.com/user-attachments/assets/97eb2e96-321c-4956-8280-8e076a2bc9c4" />


## AWS Services

- AWS Lambda
- Amazon EventBridge
- AWS Cost Explorer
- Amazon S3
- Amazon Athena
- Amazon SNS
- Amazon CloudWatch
- AWS IAM

## Features

- Daily automated cost monitoring
- Multi-region resource scanning
- Service-level cost tracking
- Resource optimization checks
- Historical cost storage
- Athena SQL analysis
- Email notifications for optimization findings

## Resource Checks

- Unattached EBS volumes
- Low-utilization EC2 instances
- NAT Gateway usage
- RDS instances
- Lambda functions
- ECS services
- ELB load balancers
- S3 buckets

## Data Flow

1. EventBridge triggers the Lambda function daily.
2. Lambda retrieves monthly cost data from AWS Cost Explorer.
3. Lambda scans resources across enabled AWS regions.
4. CloudWatch metrics are used for EC2 utilization checks.
5. Reports and Athena-ready cost data are stored in S3.
6. Athena queries the historical cost data using SQL.
7. SNS sends an email when optimization findings are detected.

## Example Athena Overview

```sql
SELECT
    month,
    service,
    ROUND(cost, 6) AS cost_usd
FROM cost_history
WHERE cost <> 0
ORDER BY month, cost DESC;
```

## Security

The Lambda execution role uses IAM permissions required for cost retrieval, resource discovery, CloudWatch metrics, S3 storage, and SNS notifications.

No AWS access keys or secrets should be committed to this repository.

## Project Goal

Provide automated visibility into AWS spending and resources so administrators can identify potential cost-optimization opportunities before unnecessary usage continues.

