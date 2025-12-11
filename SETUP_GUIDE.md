# AWS Setup Guide

This guide walks you through setting up the complete AWS infrastructure for the Banking Data Warehouse project.

## Prerequisites

- AWS Account with admin access
- AWS CLI installed and configured
- Basic understanding of AWS services (S3, Glue, Redshift, IAM)

---

## Part 1: S3 Bucket Setup

### 1.1 Create S3 Bucket

```bash
# Replace with your bucket name
export BUCKET_NAME="banking-data-warehouse-$(date +%s)"
export AWS_REGION="us-east-1"

# Create bucket
aws s3 mb s3://${BUCKET_NAME} --region ${AWS_REGION}

# Create folder structure
aws s3api put-object --bucket ${BUCKET_NAME} --key raw-data/
aws s3api put-object --bucket ${BUCKET_NAME} --key processed/
```

### 1.2 Enable Versioning (Optional but Recommended)

```bash
aws s3api put-bucket-versioning \
    --bucket ${BUCKET_NAME} \
    --versioning-configuration Status=Enabled
```

### 1.3 Upload Sample Data

```bash
# After running main.py to generate CSV files
python main.py

# Upload all CSV files
aws s3 sync . s3://${BUCKET_NAME}/raw-data/ \
    --exclude "*" \
    --include "*.csv" \
    --storage-class STANDARD_IA
```

---

## Part 2: IAM Roles Setup

### 2.1 Create Glue Service Role

Create a file named `glue-trust-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "glue.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Create the role:

```bash
# Create IAM role for Glue
aws iam create-role \
    --role-name BankingGlueServiceRole \
    --assume-role-policy-document file://glue-trust-policy.json

# Attach AWS managed policy
aws iam attach-role-policy \
    --role-name BankingGlueServiceRole \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole

# Attach S3 access policy
aws iam attach-role-policy \
    --role-name BankingGlueServiceRole \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
```

### 2.2 Create Redshift Service Role

Create a file named `redshift-trust-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "redshift.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Create the role:

```bash
# Create IAM role for Redshift
aws iam create-role \
    --role-name BankingRedshiftServiceRole \
    --assume-role-policy-document file://redshift-trust-policy.json

# Attach S3 read access
aws iam attach-role-policy \
    --role-name BankingRedshiftServiceRole \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Attach Glue access
aws iam attach-role-policy \
    --role-name BankingRedshiftServiceRole \
    --policy-arn arn:aws:iam::aws:policy/AWSGlueConsoleFullAccess
```

---

## Part 3: AWS Glue Setup

### 3.1 Create Glue Database

```bash
aws glue create-database \
    --database-input '{
        "Name": "dev_bronze",
        "Description": "Bronze layer database for raw banking data"
    }'
```

### 3.2 Create Glue Crawler

```bash
# Get your account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create crawler
aws glue create-crawler \
    --name banking-data-crawler \
    --role arn:aws:iam::${ACCOUNT_ID}:role/BankingGlueServiceRole \
    --database-name dev_bronze \
    --targets "{
        \"S3Targets\": [
            {
                \"Path\": \"s3://${BUCKET_NAME}/raw-data/\"
            }
        ]
    }" \
    --schema-change-policy "{
        \"UpdateBehavior\": \"UPDATE_IN_DATABASE\",
        \"DeleteBehavior\": \"LOG\"
    }"
```

### 3.3 Run the Crawler

```bash
# Start crawler
aws glue start-crawler --name banking-data-crawler

# Check crawler status
aws glue get-crawler --name banking-data-crawler \
    --query 'Crawler.State' --output text

# Wait for completion (State should be READY)
# This may take 5-10 minutes depending on data size
```

### 3.4 Verify Tables Created

```bash
# List tables in the database
aws glue get-tables \
    --database-name dev_bronze \
    --query 'TableList[*].Name' \
    --output table
```

Expected tables:
- `dim_customers`
- `dim_dates`
- `dim_channels`
- `dim_accounts`
- `dim_transaction_types`
- `dim_locations`
- `dim_currencies`
- `dim_investment_types`
- `dim_loans`
- `fact_transactions`
- `fact_investments`
- `fact_loans`
- `fact_customer_interactions`
- `fact_daily_balances`

---

## Part 4: Redshift Cluster Setup

### 4.1 Create Redshift Subnet Group

```bash
# Get default VPC
VPC_ID=$(aws ec2 describe-vpcs \
    --filters "Name=isDefault,Values=true" \
    --query 'Vpcs[0].VpcId' \
    --output text)

# Get subnets in the VPC
SUBNET_IDS=$(aws ec2 describe-subnets \
    --filters "Name=vpc-id,Values=${VPC_ID}" \
    --query 'Subnets[0:2].SubnetId' \
    --output text)

# Create subnet group
aws redshift create-cluster-subnet-group \
    --cluster-subnet-group-name banking-subnet-group \
    --description "Subnet group for banking data warehouse" \
    --subnet-ids ${SUBNET_IDS}
```

### 4.2 Create Security Group

```bash
# Create security group
SG_ID=$(aws ec2 create-security-group \
    --group-name banking-redshift-sg \
    --description "Security group for Redshift cluster" \
    --vpc-id ${VPC_ID} \
    --query 'GroupId' \
    --output text)

# Allow Redshift access from your IP (replace with your IP)
YOUR_IP=$(curl -s https://checkip.amazonaws.com)
aws ec2 authorize-security-group-ingress \
    --group-id ${SG_ID} \
    --protocol tcp \
    --port 5439 \
    --cidr ${YOUR_IP}/32

# Optional: Allow access from within VPC
aws ec2 authorize-security-group-ingress \
    --group-id ${SG_ID} \
    --protocol tcp \
    --port 5439 \
    --source-group ${SG_ID}
```

### 4.3 Create Redshift Cluster

```bash
# Get the IAM role ARN
REDSHIFT_ROLE_ARN=$(aws iam get-role \
    --role-name BankingRedshiftServiceRole \
    --query 'Role.Arn' \
    --output text)

# Create cluster (this takes 5-10 minutes)
aws redshift create-cluster \
    --cluster-identifier banking-dw-cluster \
    --node-type dc2.large \
    --number-of-nodes 2 \
    --master-username awsuser \
    --master-user-password 'YourSecurePassword123!' \
    --db-name dev \
    --cluster-subnet-group-name banking-subnet-group \
    --vpc-security-group-ids ${SG_ID} \
    --iam-roles ${REDSHIFT_ROLE_ARN} \
    --publicly-accessible \
    --encrypted \
    --tags Key=Project,Value=BankingDW Key=Environment,Value=Development
```

### 4.4 Wait for Cluster to be Available

```bash
# Check cluster status
aws redshift describe-clusters \
    --cluster-identifier banking-dw-cluster \
    --query 'Clusters[0].ClusterStatus' \
    --output text

# Get cluster endpoint (use this for dbt connection)
aws redshift describe-clusters \
    --cluster-identifier banking-dw-cluster \
    --query 'Clusters[0].Endpoint.Address' \
    --output text
```

---

## Part 5: Redshift Configuration

### 5.1 Connect to Redshift

Use your SQL client (DBeaver, pgAdmin, psql) with these details:
- **Host**: Output from the command above
- **Port**: 5439
- **Database**: dev
- **Username**: awsuser
- **Password**: YourSecurePassword123!

### 5.2 Create Schemas

```sql
-- Connect to Redshift and run:
CREATE SCHEMA IF NOT EXISTS dev_bronze;
CREATE SCHEMA IF NOT EXISTS dev_silver;
CREATE SCHEMA IF NOT EXISTS dev_gold;

-- Grant permissions
GRANT ALL ON SCHEMA dev_bronze TO awsuser;
GRANT ALL ON SCHEMA dev_silver TO awsuser;
GRANT ALL ON SCHEMA dev_gold TO awsuser;
```

### 5.3 Create External Schema

Replace `YOUR_ACCOUNT_ID` with your AWS account ID:

```sql
CREATE EXTERNAL SCHEMA bronze_external
FROM DATA CATALOG
DATABASE 'dev_bronze'
IAM_ROLE 'arn:aws:iam::YOUR_ACCOUNT_ID:role/BankingRedshiftServiceRole'
REGION 'us-east-1'
CREATE EXTERNAL DATABASE IF NOT EXISTS;
```

### 5.4 Verify External Tables

```sql
-- List external tables
SELECT * FROM SVV_EXTERNAL_TABLES 
WHERE schemaname = 'bronze_external';

-- Query a sample table
SELECT * FROM bronze_external.dim_customers LIMIT 10;
```

---

## Part 6: DBeaver Setup

### 6.1 Download and Install DBeaver
- Download from: https://dbeaver.io/download/
- Install for your operating system

### 6.2 Create Redshift Connection

1. Open DBeaver
2. Click **Database** → **New Database Connection**
3. Select **Amazon Redshift**
4. Enter connection details:
   - **Host**: Your Redshift endpoint
   - **Port**: 5439
   - **Database**: dev
   - **Username**: awsuser
   - **Password**: YourSecurePassword123!
5. Click **Test Connection**
6. Click **Finish**

### 6.3 Verify Connection

Run these queries to verify:

```sql
-- Check schemas
SELECT schema_name 
FROM information_schema.schemata 
WHERE schema_name LIKE 'dev%' OR schema_name = 'bronze_external';

-- Check tables in bronze_external
SELECT tablename 
FROM pg_tables 
WHERE schemaname = 'bronze_external';

-- Sample query
SELECT 
    c.customer_id,
    c.first_name,
    c.last_name,
    COUNT(ft.transaction_id) as transaction_count
FROM bronze_external.dim_customers c
LEFT JOIN bronze_external.fact_transactions ft 
    ON c.customer_id = ft.account_id
GROUP BY 1, 2, 3
LIMIT 10;
```

---

## Part 7: dbt Setup

### 7.1 Install dbt

```bash
pip install dbt-redshift
```

### 7.2 Configure dbt Profile

Edit `~/.dbt/profiles.yml`:

```yaml
dbt_redshift_dw:
  outputs:
    dev:
      type: redshift
      host: <YOUR_REDSHIFT_ENDPOINT>
      port: 5439
      user: awsuser
      password: YourSecurePassword123!
      dbname: dev
      schema: dev_gold
      threads: 4
      keepalives_idle: 240
  target: dev
```

### 7.3 Test dbt Connection

```bash
cd /path/to/dbt_redshift_dw
dbt debug
```

Expected output:
```
Connection test: OK connection ok
```

### 7.4 Run dbt

```bash
# Run all models
dbt run

# Run tests
dbt test

# Generate documentation
dbt docs generate
dbt docs serve
```

---

## Part 8: Cost Optimization

### 8.1 Pause Redshift Cluster When Not in Use

```bash
# Pause cluster
aws redshift pause-cluster --cluster-identifier banking-dw-cluster

# Resume cluster
aws redshift resume-cluster --cluster-identifier banking-dw-cluster
```

### 8.2 Set Up Budget Alerts

```bash
# Create budget to track costs
aws budgets create-budget \
    --account-id ${ACCOUNT_ID} \
    --budget file://budget.json
```

**budget.json:**
```json
{
  "BudgetName": "BankingDWBudget",
  "BudgetLimit": {
    "Amount": "50",
    "Unit": "USD"
  },
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST"
}
```

---

## Part 9: Cleanup (When Done)

### 9.1 Delete Redshift Cluster

```bash
aws redshift delete-cluster \
    --cluster-identifier banking-dw-cluster \
    --skip-final-cluster-snapshot
```

### 9.2 Delete Glue Resources

```bash
aws glue delete-crawler --name banking-data-crawler
aws glue delete-database --name dev_bronze
```

### 9.3 Delete S3 Bucket

```bash
# Empty bucket first
aws s3 rm s3://${BUCKET_NAME} --recursive

# Delete bucket
aws s3 rb s3://${BUCKET_NAME}
```

### 9.4 Delete IAM Roles

```bash
# Detach policies
aws iam detach-role-policy \
    --role-name BankingGlueServiceRole \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole

# Delete roles
aws iam delete-role --role-name BankingGlueServiceRole
aws iam delete-role --role-name BankingRedshiftServiceRole
```

---

## Troubleshooting

### Issue: Cannot connect to Redshift

**Solution:**
1. Check security group allows your IP on port 5439
2. Verify cluster is in "available" state
3. Check VPC settings allow public access

### Issue: Glue Crawler fails

**Solution:**
1. Verify IAM role has S3 read permissions
2. Check S3 path exists and contains CSV files
3. Review CloudWatch logs for Glue Crawler

### Issue: dbt connection fails

**Solution:**
1. Run `dbt debug` to see detailed error
2. Verify profiles.yml has correct credentials
3. Test connection with psql or DBeaver first

### Issue: External schema not showing tables

**Solution:**
1. Verify Glue Crawler ran successfully
2. Check IAM role attached to Redshift has Glue permissions
3. Run: `SELECT * FROM SVV_EXTERNAL_TABLES;`

---

## Estimated Costs

Based on us-east-1 pricing (as of 2024):

| Service | Configuration | Monthly Cost (Approx) |
|---------|--------------|----------------------|
| Redshift | dc2.large x 2 nodes, 24/7 | $360 |
| S3 | 10GB Standard-IA | $0.13 |
| Glue | 10 DPU-Hours/month | $4.40 |
| Data Transfer | Minimal within region | $0.50 |
| **Total** | | **~$365/month** |

**Cost Savings Tips:**
- Pause Redshift when not in use (saves ~$12/hour)
- Use Reserved Instances for long-term (save 30-50%)
- Schedule crawler to run only when needed
- Use S3 lifecycle policies to move old data to Glacier

---

## Next Steps

1. ✅ Complete AWS setup
2. ✅ Generate and upload data
3. ✅ Run dbt transformations
4. ✅ Create sample analyses in DBeaver
5. 📊 Build dashboards (Tableau/Power BI)
6. 🔄 Set up CI/CD pipeline
7. 📈 Add monitoring and alerting

---

## Additional Resources

- [AWS Redshift Documentation](https://docs.aws.amazon.com/redshift/)
- [AWS Glue Documentation](https://docs.aws.amazon.com/glue/)
- [dbt Documentation](https://docs.getdbt.com/)
- [DBeaver User Guide](https://dbeaver.com/docs/)
