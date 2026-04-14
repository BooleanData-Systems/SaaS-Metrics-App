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

| Table | Required Columns |
|-------|-----------------|
| **SUBSCRIPTIONS** (required) | SUBSCRIPTION_ID, CUSTOMER_ID, MRR_AMOUNT, EVENT_TYPE, EVENT_DATE, STATUS |
| **CUSTOMERS** (required) | CUSTOMER_ID, SEGMENT, ACQUIRED_DATE |
| **INVOICES** (optional) | INVOICE_ID, CUSTOMER_ID, INVOICE_DATE, AMOUNT |
| **USAGE_EVENTS** (optional) | CUSTOMER_ID, EVENT_NAME, EVENT_TIMESTAMP |

## Setup Instructions

1. **Install the app** from the Marketplace listing.
2. **Grant privileges** when prompted:
   - CREATE DATABASE — the app creates SAAS_METRICS_DB for pipeline layers
   - CREATE WAREHOUSE — the app creates analytics and Cortex AI warehouses
   - Cortex AI access is granted automatically via the `cortex_user` database role
3. **Bind references** — the app will prompt you to grant access to your source tables through the standard Snowsight permissions flow.
4. **Open the Streamlit app** — your granted tables are automatically detected. Click **Build SaaS Metrics**.

## AI Features (Optional)

The following features require Cortex AI access in your account:
- **AI Executive Summary** — uses SNOWFLAKE.CORTEX.COMPLETE (llama3.1-70b)
- **Churn Risk Analysis** — uses SNOWFLAKE.CORTEX.COMPLETE (llama3.1-70b)
- **Natural Language Data Chat** — uses SNOWFLAKE.CORTEX.COMPLETE (llama3.1-70b)

These features use the `cortex_user` Snowflake database role, which is automatically
granted to the app via the manifest.
If Cortex is not available in your region, the dashboard tabs still work — only the AI tab is affected.

## Data Quality

The pipeline includes automatic data quality checks:
- Completeness checks (NOT NULL on key columns)
- Uniqueness checks (deduplication on primary keys)
- Validity checks (positive MRR, valid date ranges)
- Circuit breaker pattern (halts pipeline if failure rate exceeds 5%)
- Quarantine table for rejected records

## Support

For issues or questions, contact Boolean Data at support@booleandata.io
