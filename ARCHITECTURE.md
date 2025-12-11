# Architecture Documentation

## System Architecture

This document provides detailed architecture diagrams for the Banking Data Warehouse project.

## Data Flow Architecture

```mermaid
graph TB
    A[Python Script<br/>main.py] -->|Generate CSV| B[Local Files]
    B -->|Upload| C[AWS S3 Bucket<br/>Bronze Layer]
    C -->|Catalog| D[AWS Glue Crawler]
    D -->|Metadata| E[AWS Glue Catalog]
    E -->|External Tables| F[AWS Redshift Cluster]
    F -->|dbt Run| G[Silver Layer<br/>Staging Tables]
    G -->|dbt Run| H[Gold Layer<br/>Dimensional Models]
    H -->|Query| I[DBeaver<br/>Analytics]
    
    style A fill:#e1f5ff
    style C fill:#fff4e1
    style F fill:#ffe1e1
    style G fill:#e1ffe1
    style H fill:#ffe1ff
    style I fill:#f0e1ff
```

## Medallion Architecture Layers

```mermaid
graph LR
    subgraph Bronze["🥉 Bronze Layer (Raw)"]
        B1[S3 CSV Files]
        B2[Glue Catalog]
        B3[External Tables]
    end
    
    subgraph Silver["🥈 Silver Layer (Staging)"]
        S1[stg_dim_customer]
        S2[stg_dim_account]
        S3[stg_fact_transaction]
        S4[stg_fact_investment]
    end
    
    subgraph Gold["🥇 Gold Layer (Business)"]
        G1[dim_customer]
        G2[dim_account]
        G3[fact_transaction]
        G4[fact_investment]
    end
    
    Bronze --> Silver
    Silver --> Gold
    
    style Bronze fill:#cd7f32,color:#fff
    style Silver fill:#c0c0c0,color:#000
    style Gold fill:#ffd700,color:#000
```

## Star Schema - Dimensional Model

```mermaid
erDiagram
    FACT_TRANSACTION ||--o{ DIM_CUSTOMER : "customer_id"
    FACT_TRANSACTION ||--o{ DIM_DATE : "date_id"
    FACT_TRANSACTION ||--o{ DIM_ACCOUNT : "account_id"
    FACT_TRANSACTION ||--o{ DIM_CHANNEL : "channel_id"
    FACT_TRANSACTION ||--o{ DIM_LOCATION : "location_id"
    FACT_TRANSACTION ||--o{ DIM_CURRENCY : "currency_id"
    FACT_TRANSACTION ||--o{ DIM_TRANSACTION_TYPE : "transaction_type_id"
    
    FACT_INVESTMENT ||--o{ DIM_ACCOUNT : "account_id"
    FACT_INVESTMENT ||--o{ DIM_DATE : "date_id"
    FACT_INVESTMENT ||--o{ DIM_LOCATION : "location_id"
    FACT_INVESTMENT ||--o{ DIM_CURRENCY : "currency_id"
    FACT_INVESTMENT ||--o{ DIM_INVESTMENT_TYPE : "investment_type_id"
    
    FACT_LOAN ||--o{ DIM_ACCOUNT : "account_id"
    FACT_LOAN ||--o{ DIM_DATE : "date_id"
    FACT_LOAN ||--o{ DIM_LOCATION : "location_id"
    FACT_LOAN ||--o{ DIM_CURRENCY : "currency_id"
    FACT_LOAN ||--o{ DIM_LOAN : "loan_id"
    
    DIM_ACCOUNT ||--o{ DIM_CUSTOMER : "customer_id"
    
    DIM_CUSTOMER {
        int customer_id PK
        string first_name
        string last_name
        string email
        string address
        string city
        string state
        string postal_code
        string phone_number
    }
    
    DIM_ACCOUNT {
        int account_id PK
        int customer_id FK
        bigint account_number
        string account_type
        decimal account_balance
        int credit_score
    }
    
    DIM_DATE {
        int date_id PK
        date date
        int day
        int month
        int year
        int weekday
    }
    
    FACT_TRANSACTION {
        int transaction_id PK
        int date_id FK
        int transaction_type_id FK
        int account_id FK
        int channel_id FK
        int location_id FK
        int currency_id FK
        decimal transaction_amount
        string transaction_status
    }
    
    FACT_INVESTMENT {
        int investment_id PK
        int date_id FK
        int investment_type_id FK
        int account_id FK
        int location_id FK
        int currency_id FK
        decimal investment_amount
        decimal investment_return
    }
    
    FACT_LOAN {
        int loan_fact_id PK
        int date_id FK
        int loan_id FK
        int account_id FK
        int location_id FK
        int currency_id FK
        decimal loan_amount
        string loan_status
    }
```

## dbt Project Structure

```mermaid
graph TD
    subgraph Sources["📁 Sources (Bronze)"]
        SRC1[bronze_external.dim_customers]
        SRC2[bronze_external.dim_accounts]
        SRC3[bronze_external.fact_transactions]
        SRC4[bronze_external.fact_investments]
    end
    
    subgraph Silver["🥈 Silver Models (dev_silver)"]
        STG1[stg_dim_customer]
        STG2[stg_dim_account]
        STG3[stg_fact_transaction]
        STG4[stg_fact_investment]
    end
    
    subgraph Gold["🥇 Gold Models (dev_gold)"]
        DIM1[dim_customer]
        DIM2[dim_account]
        FACT1[fact_transaction]
        FACT2[fact_investment]
    end
    
    SRC1 --> STG1
    SRC2 --> STG2
    SRC3 --> STG3
    SRC4 --> STG4
    
    STG1 --> DIM1
    STG2 --> DIM2
    STG3 --> FACT1
    STG4 --> FACT2
    
    style Sources fill:#fff4e1
    style Silver fill:#e1ffe1
    style Gold fill:#ffe1ff
```

## AWS Infrastructure

```mermaid
graph TB
    subgraph AWS["AWS Cloud"]
        subgraph Storage["Storage Layer"]
            S3[S3 Bucket<br/>banking-data]
        end
        
        subgraph Catalog["Catalog Layer"]
            GLUE[Glue Crawler]
            CAT[Glue Data Catalog]
        end
        
        subgraph Compute["Compute Layer"]
            RS[Redshift Cluster<br/>dc2.large x 2 nodes]
            SPEC[Redshift Spectrum]
        end
        
        subgraph Security["Security"]
            IAM[IAM Roles]
            VPC[VPC]
            SG[Security Groups]
        end
    end
    
    S3 --> GLUE
    GLUE --> CAT
    CAT --> SPEC
    SPEC --> RS
    IAM --> RS
    VPC --> RS
    SG --> RS
    
    style S3 fill:#ff9900
    style RS fill:#8c4fff
    style GLUE fill:#cc2264
```

## Data Pipeline Execution Flow

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant Py as Python Script
    participant S3 as AWS S3
    participant Glue as Glue Crawler
    participant RS as Redshift
    participant DBT as dbt
    participant DB as DBeaver
    
    Dev->>Py: python main.py
    Py->>Py: Generate synthetic data
    Py->>S3: Upload CSV files
    
    Dev->>Glue: Run crawler
    Glue->>S3: Scan files
    Glue->>Glue: Create schema
    Glue->>RS: Catalog metadata
    
    Dev->>DBT: dbt run
    DBT->>RS: Create Silver tables
    DBT->>RS: Create Gold tables
    DBT->>RS: Run tests
    DBT-->>Dev: Success ✓
    
    Dev->>DB: Query analytics
    DB->>RS: Execute SQL
    RS-->>DB: Return results
    DB-->>Dev: Display insights
```

## Deployment Architecture

```mermaid
graph TB
    subgraph Dev["Development Environment"]
        LOCAL[Local Development]
        GIT[Git Repository]
    end
    
    subgraph CI["CI/CD Pipeline (Future)"]
        GHA[GitHub Actions]
        TEST[Run Tests]
        LINT[SQL Linting]
    end
    
    subgraph Prod["Production AWS"]
        S3P[S3 Production]
        RSP[Redshift Production]
        DBTP[dbt Production]
    end
    
    LOCAL --> GIT
    GIT --> GHA
    GHA --> TEST
    TEST --> LINT
    LINT --> DBTP
    DBTP --> S3P
    S3P --> RSP
    
    style Dev fill:#e1f5ff
    style CI fill:#ffe1e1
    style Prod fill:#e1ffe1
```

## Security Architecture

```mermaid
graph TD
    subgraph Public["Public Internet"]
        USER[Data Engineer]
    end
    
    subgraph VPC["AWS VPC"]
        subgraph Public_Subnet["Public Subnet"]
            NAT[NAT Gateway]
        end
        
        subgraph Private_Subnet["Private Subnet"]
            RS[Redshift Cluster]
            S3GW[S3 Gateway Endpoint]
        end
        
        subgraph Security_Layer["Security Layer"]
            SG[Security Groups]
            NACL[Network ACLs]
            IAM[IAM Roles]
        end
    end
    
    subgraph Services["AWS Services"]
        S3[S3 Bucket]
        GLUE[Glue Service]
    end
    
    USER -->|HTTPS| RS
    RS --> S3GW
    S3GW --> S3
    GLUE --> S3
    IAM --> RS
    SG --> RS
    NACL --> Private_Subnet
    
    style Public fill:#ff9999
    style Private_Subnet fill:#99ff99
    style Security_Layer fill:#ffff99
```

---

## Key Design Decisions

### 1. Medallion Architecture
- **Bronze**: Raw data preservation with minimal transformation
- **Silver**: Cleaned, validated, and enriched data
- **Gold**: Business-ready aggregated data

### 2. Star Schema Design
- Optimized for analytical queries
- Denormalized for query performance
- SCD Type 2 for historical tracking

### 3. Incremental Loading
- Reduces processing time for large datasets
- Updates only changed/new records
- Maintains audit trail with dbt

### 4. AWS Redshift Choice
- Columnar storage for analytics
- MPP architecture for parallel processing
- Seamless S3 integration

### 5. dbt for Transformation
- SQL-first approach
- Built-in testing framework
- Version-controlled transformations
- Documentation generation
