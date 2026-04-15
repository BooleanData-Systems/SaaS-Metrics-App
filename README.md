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
   - **CREATE DATABASE** — the app creates SAAS_METRICS_DB for pipeline layers
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
| register_reference | Handles reference ADD/REMOVE/CLEAR callbacks |
| initialize_pipeline | Creates all schemas and DQ tables in the target database |
| describe_reference | Reads column names from a bound reference using reference() |
| load_reference_to_staging | Copies data from a bound reference into STAGING using reference() |

### Application Roles

| Role | Access |
|------|--------|
| app_public | Base role — Streamlit app, procedures, reference binding |
| app_admin | Inherits app_public — can initialize pipeline |

## Data Quality

The pipeline includes automatic data quality checks:
- **Completeness** checks (NOT NULL on key columns)
- **Uniqueness** checks (deduplication on primary keys)
- **Validity** checks (positive MRR, valid date ranges)
- **Circuit breaker** pattern (halts pipeline if failure rate exceeds 5%)
- **Quarantine** table for rejected records

## AI Features (Optional)

The following features require Cortex AI access in your account:
- **AI Executive Summary** — uses SNOWFLAKE.CORTEX.COMPLETE (mistral-large2)
- **Churn Risk Analysis** — uses SNOWFLAKE.CORTEX.COMPLETE (mistral-large2)
- **Natural Language Data Chat** — uses SNOWFLAKE.CORTEX.COMPLETE (mistral-large2)

Cortex AI access is available by default via the CORTEX_USER database role, which is granted to PUBLIC in all Snowflake accounts.

If your account has revoked the CORTEX_USER role from PUBLIC, run:

    GRANT DATABASE ROLE SNOWFLAKE.CORTEX_USER TO APPLICATION <your_app_name>;

If Cortex is not available in your region, the dashboard tabs still work — only the AI tab is affected.

## Support

For issues or questions, contact Boolean Data at support@booleandata.io

