# SaaS Metrics Accelerator

Instantly transform raw SaaS data into actionable revenue, churn, and customer
insights through an automated medallion pipeline — all within Snowflake.

## What This App Does

The SaaS Metrics Accelerator ingests your existing subscription and customer
data and builds a full **RAW → BRONZE → SILVER → GOLD → SEMANTIC** pipeline
that computes industry-standard SaaS KPIs:

| Category | Metrics |
|----------|---------|
| Revenue | ARR, MRR, Net New MRR, Expansion/Contraction MRR, ARPU |
| Churn | Gross Churn Rate, Net Churn Rate, Revenue Churn Rate, Cohort Retention |
| Customers | LTV, LTV Tiers (Platinum/Gold/Silver/Bronze), Tenure, Industry Breakdown |
| Health | Quick Ratio, Subscription Health by Plan Tier |
| Usage | Feature Adoption, Engagement Heatmaps, Session Analytics |
| AI | Executive Summary, Churn Risk Prediction, Natural Language Chat |

## Required Tables

| Table | Required? | Required Columns |
|-------|-----------|-----------------|
| **SUBSCRIPTIONS** | Yes | SUBSCRIPTION_ID, CUSTOMER_ID, MRR_AMOUNT, EVENT_TYPE, EVENT_DATE, STATUS |
| **CUSTOMERS** | Yes | CUSTOMER_ID, SEGMENT, ACQUIRED_DATE |
| **INVOICES** | No | INVOICE_ID, CUSTOMER_ID, INVOICE_DATE, AMOUNT |
| **USAGE_EVENTS** | No | CUSTOMER_ID, EVENT_NAME, EVENT_TIMESTAMP |

## Consumer Setup (Zero SQL Required)

1. **Install the app** from the Marketplace listing.
2. **Grant privileges** when prompted in Snowsight:
   - **CREATE DATABASE** — the app creates `SAAS_METRICS_DB` for pipeline layers
   - **CREATE WAREHOUSE** — the app creates analytics and Cortex AI warehouses
3. **Bind table references** — the app sidebar prompts you to grant access to your source tables. Click **Grant Subscriptions** and **Grant Customers**, then pick your tables from the dropdown.
4. **Click "Build SaaS Metrics"** — the pipeline runs automatically.

No manual SQL or GRANT statements are needed. Everything is handled through the Snowsight permission flow.

## What the App Creates

### Database: SAAS_METRICS_DB

| Schema | Purpose |
|--------|---------|
| STAGING | Raw VARIANT ingestion from source references |
| BRONZE | Flattened relational tables from VARIANT |
| SILVER | Cleansed, deduplicated, null-handled tables |
| GOLD | Star schema (DIM_CUSTOMERS, FACT_SUBSCRIPTIONS, FACT_REVENUE, FACT_USAGE, FACT_MONTHLY_METRICS) |
| SEMANTIC | Secure views for KPIs (V_ARR_OVERVIEW, V_CHURN_ANALYSIS, V_CUSTOMER_LTV, V_SUBSCRIPTION_HEALTH, V_FEATURE_ADOPTION, V_REVENUE_TRENDS) |
| AUDIT | Data quality check log (DQ_CHECK_LOG) |
| QUARANTINE | Rejected records and circuit breaker log |

### Stored Procedures

| Procedure | Purpose |
|-----------|---------|
| `register_reference` | Handles reference ADD/REMOVE/CLEAR callbacks |
| `initialize_pipeline` | Creates all schemas and DQ tables in the target database |
| `describe_reference` | Reads column names from a bound reference using `reference()` |
| `load_reference_to_staging` | Copies data from a bound reference into STAGING using `reference()` |

### Application Roles

| Role | Access |
|------|--------|
| `app_public` | Base role — Streamlit app, procedures, reference binding |
| `app_admin` | Inherits `app_public` — can initialize pipeline |

## Data Quality

The pipeline includes automatic data quality checks:
- **Completeness** checks (NOT NULL on key columns)
- **Uniqueness** checks (deduplication on primary keys)
- **Validity** checks (positive MRR, valid date ranges)
- **Circuit breaker** pattern (halts pipeline if failure rate exceeds 5%)
- **Quarantine** table for rejected records

## AI Features (Optional)

The following features require Cortex AI access in your account:
- **AI Executive Summary** — uses `SNOWFLAKE.CORTEX.COMPLETE` (mistral-large2)
- **Churn Risk Analysis** — uses `SNOWFLAKE.CORTEX.COMPLETE` (mistral-large2)
- **Natural Language Data Chat** — uses `SNOWFLAKE.CORTEX.COMPLETE` (mistral-large2)

Cortex AI access is available by default via the `CORTEX_USER` database role, which is granted to `PUBLIC` in all Snowflake accounts.

If your account has revoked the `CORTEX_USER` role from `PUBLIC`, run:

```sql
GRANT DATABASE ROLE SNOWFLAKE.CORTEX_USER TO APPLICATION <your_app_name>;

**Sample Test Data**
**To create sample source data for testing:**
USE ROLE ACCOUNTADMIN;

CREATE DATABASE IF NOT EXISTS SAAS_SOURCE_DATA;
CREATE SCHEMA IF NOT EXISTS SAAS_SOURCE_DATA.RAW;

CREATE OR REPLACE TABLE SAAS_SOURCE_DATA.RAW.CUSTOMERS (
    CUSTOMER_ID VARCHAR(50), SEGMENT VARCHAR(100), ACQUIRED_DATE DATE,
    CUSTOMER_NAME VARCHAR(200), EMAIL VARCHAR(300), INDUSTRY VARCHAR(100),
    COMPANY_SIZE VARCHAR(50), COUNTRY VARCHAR(100)
);
INSERT INTO SAAS_SOURCE_DATA.RAW.CUSTOMERS
SELECT 'CUST-' || LPAD(SEQ4()::VARCHAR, 4, '0'),
    CASE MOD(SEQ4(), 4) WHEN 0 THEN 'Enterprise' WHEN 1 THEN 'Mid-Market' WHEN 2 THEN 'SMB' ELSE 'Startup' END,
    DATEADD('DAY', -UNIFORM(90, 900, RANDOM()), CURRENT_DATE()),
    'Company ' || SEQ4(), 'contact' || SEQ4() || '@company' || SEQ4() || '.com',
    CASE MOD(SEQ4(), 6) WHEN 0 THEN 'Technology' WHEN 1 THEN 'Healthcare' WHEN 2 THEN 'Finance' WHEN 3 THEN 'Retail' WHEN 4 THEN 'Manufacturing' ELSE 'Education' END,
    CASE MOD(SEQ4(), 4) WHEN 0 THEN '1-50' WHEN 1 THEN '51-200' WHEN 2 THEN '201-1000' ELSE '1000+' END,
    CASE MOD(SEQ4(), 5) WHEN 0 THEN 'US' WHEN 1 THEN 'UK' WHEN 2 THEN 'Germany' WHEN 3 THEN 'Australia' ELSE 'Canada' END
FROM TABLE(GENERATOR(ROWCOUNT => 50));

CREATE OR REPLACE TABLE SAAS_SOURCE_DATA.RAW.SUBSCRIPTIONS (
    SUBSCRIPTION_ID VARCHAR(50), CUSTOMER_ID VARCHAR(50), MRR_AMOUNT NUMBER(18,2),
    EVENT_TYPE VARCHAR(50), EVENT_DATE DATE, STATUS VARCHAR(50),
    PLAN_NAME VARCHAR(100), PLAN_TIER VARCHAR(50), BILLING_CYCLE VARCHAR(20),
    START_DATE DATE, END_DATE DATE
);
INSERT INTO SAAS_SOURCE_DATA.RAW.SUBSCRIPTIONS
SELECT 'SUB-' || LPAD(SEQ4()::VARCHAR, 5, '0'),
    'CUST-' || LPAD(MOD(SEQ4(), 50)::VARCHAR, 4, '0'),
    CASE MOD(SEQ4(), 5) WHEN 0 THEN 49.00 WHEN 1 THEN 99.00 WHEN 2 THEN 199.00 WHEN 3 THEN 499.00 ELSE 999.00 END,
    CASE MOD(SEQ4(), 5) WHEN 0 THEN 'new' WHEN 1 THEN 'upgrade' WHEN 2 THEN 'new' WHEN 3 THEN 'downgrade' ELSE 'churn' END,
    DATEADD('DAY', -UNIFORM(1, 730, RANDOM()), CURRENT_DATE()),
    CASE MOD(SEQ4(), 5) WHEN 4 THEN 'churned' ELSE 'active' END,
    CASE MOD(SEQ4(), 3) WHEN 0 THEN 'Starter' WHEN 1 THEN 'Professional' ELSE 'Enterprise' END,
    CASE MOD(SEQ4(), 3) WHEN 0 THEN 'Basic' WHEN 1 THEN 'Pro' ELSE 'Premium' END,
    CASE MOD(SEQ4(), 2) WHEN 0 THEN 'monthly' ELSE 'annual' END,
    DATEADD('DAY', -UNIFORM(30, 730, RANDOM()), CURRENT_DATE()),
    CASE WHEN MOD(SEQ4(), 5) = 4 THEN DATEADD('DAY', -UNIFORM(1, 30, RANDOM()), CURRENT_DATE()) ELSE NULL END
FROM TABLE(GENERATOR(ROWCOUNT => 200));

CREATE OR REPLACE TABLE SAAS_SOURCE_DATA.RAW.INVOICES (
    INVOICE_ID VARCHAR(50), CUSTOMER_ID VARCHAR(50), INVOICE_DATE DATE,
    AMOUNT NUMBER(18,2), TAX_AMOUNT NUMBER(18,2), TOTAL_AMOUNT NUMBER(18,2),
    PAYMENT_STATUS VARCHAR(50)
);
INSERT INTO SAAS_SOURCE_DATA.RAW.INVOICES
SELECT 'INV-' || LPAD(SEQ4()::VARCHAR, 6, '0'),
    'CUST-' || LPAD(MOD(SEQ4(), 50)::VARCHAR, 4, '0'),
    DATEADD('DAY', -UNIFORM(1, 365, RANDOM()), CURRENT_DATE()),
    UNIFORM(49, 999, RANDOM())::NUMBER(18,2), UNIFORM(5, 100, RANDOM())::NUMBER(18,2),
    UNIFORM(54, 1099, RANDOM())::NUMBER(18,2),
    CASE MOD(SEQ4(), 10) WHEN 0 THEN 'overdue' ELSE 'paid' END
FROM TABLE(GENERATOR(ROWCOUNT => 300));

CREATE OR REPLACE TABLE SAAS_SOURCE_DATA.RAW.USAGE_EVENTS (
    CUSTOMER_ID VARCHAR(50), EVENT_NAME VARCHAR(200), EVENT_TIMESTAMP TIMESTAMP_NTZ,
    SESSION_ID VARCHAR(100), FEATURE_NAME VARCHAR(200), DURATION_SECONDS NUMBER
);
INSERT INTO SAAS_SOURCE_DATA.RAW.USAGE_EVENTS
SELECT 'CUST-' || LPAD(MOD(SEQ4(), 50)::VARCHAR, 4, '0'),
    CASE MOD(SEQ4(), 8) WHEN 0 THEN 'page_view' WHEN 1 THEN 'report_generated' WHEN 2 THEN 'dashboard_opened' WHEN 3 THEN 'export_csv' WHEN 4 THEN 'api_call' WHEN 5 THEN 'user_invited' WHEN 6 THEN 'integration_added' ELSE 'settings_changed' END,
    DATEADD('SECOND', -UNIFORM(1, 2592000, RANDOM()), CURRENT_TIMESTAMP()),
    'SESS-' || LPAD(UNIFORM(1, 5000, RANDOM())::VARCHAR, 6, '0'),
    CASE MOD(SEQ4(), 6) WHEN 0 THEN 'Analytics' WHEN 1 THEN 'Reporting' WHEN 2 THEN 'Integrations' WHEN 3 THEN 'User Management' WHEN 4 THEN 'API' ELSE 'Dashboard' END,
    UNIFORM(5, 600, RANDOM())
FROM TABLE(GENERATOR(ROWCOUNT => 1000));

USE ROLE ACCOUNTADMIN;
GRANT USAGE ON DATABASE SAAS_SOURCE_DATA TO APPLICATION <your_app_name>;
GRANT USAGE ON SCHEMA SAAS_SOURCE_DATA.RAW TO APPLICATION <your_app_name>;
GRANT SELECT ON ALL TABLES IN SCHEMA SAAS_SOURCE_DATA.RAW TO APPLICATION <your_app_name>;
GRANT DATABASE ROLE SNOWFLAKE.CORTEX_USER TO APPLICATION <your_app_name>;
Then open the Streamlit app, grant the table references via the sidebar, and click Build SaaS Metrics.

**Support**
For issues or questions, contact Boolean Data at support@booleandata.io
