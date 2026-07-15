# Banking Data Governance and Sales Reporting in Microsoft Fabric and Power BI

## Overview

This portfolio project demonstrates how **data governance, data quality controls, master data management, governed business definitions, and management reporting** can be implemented in Microsoft Fabric.

The solution combines two related business domains:

1. **Legal Entity Data Governance**
2. **Banking Sales Reporting Governance**

The project shows how raw source data is standardized, validated, deduplicated, consolidated into trusted Gold-layer datasets, exposed through Power BI reports and Fabric App for both governance teams and business users.

---

## Business Problem

Banks often use several systems to manage the same legal entity or customer information. CRM, KYC, registry, and sales systems may contain:

- duplicate records;
- missing identifiers;
- conflicting statuses;
- invalid reference values;
- inconsistent product classifications;
- incomplete activation data;
- contracts that should not be included in management reporting.

Without governed definitions and data quality controls, these issues can distort customer master data and sales KPIs.

This project addresses that problem by creating:

- a trusted **Golden Record** for legal entities;
- a deduplicated banking product sales fact table;
- a governed sales definition;
- a centralized data quality rule catalogue;
- a monthly sales KPI mart;
- Power BI reports for data quality monitoring and sales performance.

---

## Solution Architecture

```text
CSV source files
      |
      v
Microsoft Fabric Lakehouse
      |
      +-- Bronze
      |     Raw source-aligned tables
      |
      +-- Silver
      |     Standardized and typed tables
      |     Source defects intentionally preserved
      |
      +-- Gold
      |     Golden legal entity dimension
      |     Deduplicated banking sales fact
      |     Monthly sales KPI mart
      |
      +-- Governance
            Data quality rule results
      |
      v
Power BI semantic models and reports
      |
      v
Microsoft Fabric App: Data Quality Management Portal 

```

The Fabric pipeline executes the notebooks in sequence:

```text
01_load_bronze_tables
        |
        v
02_create_silver_tables
        |
        v
03_create_gold_tables
        |
        v
04_run_data_quality_checks
```
<img width="2068" height="516" alt="image" src="https://github.com/user-attachments/assets/5da08ea8-d64d-4f1d-91e3-1d30f99ab1ff" />


---

## Technology Stack

- Microsoft Fabric
- Fabric Lakehouse
- PySpark
- Delta tables
- Fabric Data Pipeline
- Fabric Apps
- Power BI
- DAX
- GitHub
- Microsoft Purview planned as the governance catalogue layer

---

## Dataset

The project uses synthetic banking data with intentionally introduced data quality defects.

| Dataset | Rows |
|---|---:|
| CRM legal entities | 105 |
| KYC legal entities | 100 |
| Registry legal entities | 100 |
| Customers | 100 |
| Product contracts | 1,025 |
| Sales targets | 432 |
| Branches | 6 |

Controlled data quality issues include:

- 5 duplicate CRM legal entity records;
- 25 duplicate product contracts;
- missing registry codes;
- invalid registry country codes;
- active legal entities with closure dates;
- invalid parent references;
- inconsistent KYC and registry statuses;
- active contracts without activation dates;
- invalid product categories.

The duplicate-related controls produce **30 uniqueness issue occurrences**:

```text
5 duplicate legal entity records
+ 25 duplicate contract records
= 30 uniqueness issues
```

---

## Lakehouse Structure

### Bronze Layer

The Bronze layer stores source-aligned data with minimal transformation.

```text
bronze.crm_legal_entities
bronze.kyc_legal_entities
bronze.registry_legal_entities
bronze.customers
bronze.product_contracts
bronze.sales_targets
bronze.branches
```

### Silver Layer

The Silver layer standardizes column names, types, dates, and text values.

Source defects are intentionally retained so they can be detected and measured by the data quality controls.

```text
silver.crm_legal_entities
silver.kyc_legal_entities
silver.registry_legal_entities
silver.customers
silver.product_contracts
silver.sales_targets
silver.branches
```

### Gold Layer

```text
gold.dim_legal_entity_golden
gold.fct_banking_product_sales
gold.mart_monthly_sales_kpis
```

### Governance Layer

```text
governance.dq_rule_results
```

The complete solution contains **18 managed tables**.

---

## Legal Entity Golden Record

The Golden Record consolidates registry, CRM, and KYC information into one trusted legal entity record.

### Source Authority

| Data attribute | Authoritative source |
|---|---|
| Registry code | Business Registry |
| Official legal name | Business Registry |
| Country | Business Registry |
| Legal status | Business Registry |
| Incorporation date | Business Registry |
| Closure date | Business Registry |
| Parent registry code | Business Registry |
| Customer segment | CRM |
| Relationship manager | CRM |
| CRM status | CRM |
| KYC status | KYC |
| Risk level | KYC |
| LEI | KYC |
| Last KYC review date | KYC |

### Matching Key

```text
registry_code
```

### Survivorship Rules

For duplicate CRM records:

1. prefer `Active` records;
2. prefer the newest `created_date`;
3. use the smallest CRM entity ID as a deterministic tie-breaker.

For duplicate KYC records:

1. prefer `Approved` records;
2. prefer the latest `last_review_date`;
3. use the smallest KYC entity ID as a deterministic tie-breaker.

Only records with a non-null registry code and a valid Baltic country code (`EE`, `LV`, `LT`) are included in the Golden Record.

### Result

```text
98 trusted legal entity Golden Records
```

The technical key is generated as:

```text
SHA-256(registry_code)
```

---

## Governed Sales Definition

A contract is classified as a governed sale only when all approved business conditions are met.

```text
contract_status = "Active"
AND activation_date IS NOT NULL
AND product_type is in the approved product list
```

Approved products:

- Consumer Loan
- Mortgage
- Credit Card
- Current Account
- Savings Account
- Term Deposit
- Investment Product
- Business Loan

Duplicate contract IDs are removed before Gold reporting.

### Sales Processing Result

```text
1,025 source contract rows
-   25 duplicate rows
= 1,000 unique contracts

724 governed contracts
276 excluded contracts
72.4% governed contract share
```

An excluded contract is not necessarily deleted. It remains available in the detailed fact table for investigation but is not included in governed sales KPIs.

Examples of exclusion reasons:

- contract is Cancelled, Pending, or Rejected;
- activation date is missing;
- product category is invalid.

---

## Gold Tables

### `gold.dim_legal_entity_golden`

One trusted record per eligible legal entity.

Main use cases:

- master data reporting;
- KYC and customer oversight;
- legal entity lookup;
- source reconciliation;
- future Purview glossary and lineage integration.

### `gold.fct_banking_product_sales`

One row per unique banking product contract.

Main use cases:

- governed versus excluded contract analysis;
- detailed product, customer, branch, and channel analysis;
- investigation of excluded records;
- sales amount analysis.

### `gold.mart_monthly_sales_kpis`

One row per reporting combination of:

```text
month + product type + sales channel
```

Main use cases:

- actual versus target reporting;
- monthly sales trends;
- management KPI reporting;
- product and channel performance.

The fact table and KPI mart serve different purposes:

```text
Sales fact = detailed contract-level evidence
KPI mart   = aggregated management reporting
```

---

## Data Quality Rule Catalogue

The solution implements 12 data quality rules.

### Legal Entity Rules

| Rule ID | Rule | Dimension | Severity |
|---|---|---|---|
| DQ-LE-001 | Missing CRM registry code | Completeness | Critical |
| DQ-LE-002 | Duplicate non-null CRM registry code | Uniqueness | Critical |
| DQ-LE-003 | Invalid registry country code | Validity | High |
| DQ-LE-004 | Active legal entity has a closure date | Consistency | High |
| DQ-LE-005 | Parent legal entity does not exist | Referential Integrity | Medium |
| DQ-LE-006 | Registry and KYC status inconsistency | Consistency | High |

### Sales Rules

| Rule ID | Rule | Dimension | Severity |
|---|---|---|---|
| DQ-SALES-001 | Duplicate contract ID | Uniqueness | Critical |
| DQ-SALES-002 | Active contract missing activation date | Completeness | High |
| DQ-SALES-003 | Invalid product category | Validity | High |
| DQ-SALES-004 | Missing customer reference | Referential Integrity | Critical |
| DQ-SALES-005 | Missing branch reference | Referential Integrity | High |
| DQ-SALES-006 | Cancelled contract classified as governed sale | Validity | Critical |

The rule results are written to:

```text
governance.dq_rule_results
```

Main fields:

```text
rule_id
data_domain
data_asset
rule_description
dq_dimension
severity
failed_record_count
rule_status
executed_at
```

> `failed_record_count` measures rule failure occurrences. A single physical record can fail more than one rule, so the total should not automatically be interpreted as the number of unique affected records.

---

## Power BI Reports

## 1. Data Quality Report

### Audience

- Data Stewards
- Data Owners
- Governance Teams
- Data Analysts

### Purpose

The report shows:

- which data quality rules failed;
- how many failure occurrences were detected;
- which data domain and asset are affected;
- the severity of each issue;
- the data quality dimension involved;
- which issues should be prioritized for remediation.

### Main KPIs

- Failed Rules
- Passed Rules
- Total Failed Records
- Critical Failed Rules
- DQ Pass Rate
- Average Failed Records

### Main Visuals

- Failed records by data domain
- Data quality issues by dimension
- Failed rules by severity
- Passed versus failed rules
- Detailed rule results table

<img width="1896" height="1068" alt="dashboard dq" src="https://github.com/user-attachments/assets/35cc3ad8-d22d-4375-ae0a-efb98a5c9ec2" />
---



## 2. Sales Overview

### Audience

- Sales Management
- Business Performance Teams
- Product Owners
- Business Analysts

### Purpose

The report presents governed management KPIs after the approved sales rules have been applied.

### Main KPIs

- Products Sold
- Sales Amount
- Customers with Sales
- Target Count Achievement
- Target Amount Achievement

### Main Visuals

- Monthly products sold trend
- Products sold by product type
- Products sold by sales channel
- Actual sales versus target
- Product and channel slicers

<img width="1904" height="1064" alt="dashb sales 1" src="https://github.com/user-attachments/assets/b794e0d1-9e61-438b-8c17-a24e69e92733" />

---

## 3. Governed Sales Quality

### Audience

- Data Stewards
- Sales Reporting Owners
- Business Analysts
- Product Owners

### Purpose

The report connects data governance with business impact by showing which contracts qualify for KPI reporting and which contracts are excluded.

### Main KPIs

- Governed Contracts
- Excluded Contracts
- Governed Contract Share
- Governed Sales Amount
- Governed Sales Amount Share

### Main Visuals

- Governed versus excluded contracts
- Excluded contracts by status
- Excluded contracts by product
- Contracts excluded from governed sales detail table

The detail table is filtered to:

```text
is_governed_sale = FALSE
```
<img width="1898" height="1062" alt="dashb sales 2" src="https://github.com/user-attachments/assets/13864dc3-3fae-4e03-9931-3fe2a501f236" />

---
## Miscrosoft Fabric App

<img width="2810" height="1448" alt="image" src="https://github.com/user-attachments/assets/26239531-ae67-4208-80c0-dbfc9d65f155" />

---
## Business Impact

The project demonstrates how data quality defects can affect business reporting.

Examples:

```text
Duplicate contracts
-> inflate product sales counts and sales amount

Missing activation date
-> prevents reliable confirmation that a sale occurred

Invalid product category
-> misclassifies product performance

Cancelled contract counted as a sale
-> inflates management KPIs

Duplicate legal entities
-> fragment customer and KYC information
```

The solution creates traceability from:

```text
Source defect
-> DQ rule
-> remediation priority
-> governed Gold dataset
-> trusted Power BI KPI
```

---

## Repository Structure

```text
banking-data-governance-fabric/
|
+-- README.md
+-- docs/
|   +-- Banking_Data_Governance_Business_Requirements_Specification.docx
|   +-- Banking_Data_Governance_Functional_Specification.docx
|   +-- data-quality-rule-catalogue.md
|   +-- data-dictionary.md
|   +-- dashboard-user-guide.md
|
+-- data/
|   +-- sample/
|
+-- fabric/
|   +-- notebooks/
|   +-- pipelines/
|   +-- lakehouse/
|   +-- power-bi/
|
+-- screenshots/
    +-- dq-dashboard.png
    +-- sales-overview.png
    +-- governed-sales-quality.png
```

Fabric Git integration stores supported workspace item definitions and code. It does not store the physical Lakehouse table data as a Git backup.

---

## How to Run

1. Create a Microsoft Fabric workspace backed by Fabric capacity.
2. Create a Lakehouse named:

```text
banking_governance_lakehouse
```

3. Create the following schemas:

```text
bronze
silver
gold
governance
```

4. Upload the sample CSV files under:

```text
Files/bronze/legal_entity/
Files/bronze/sales/
```

5. Add the Lakehouse to all notebooks.
6. Run the notebooks in this order:

```text
01_load_bronze_tables
02_create_silver_tables
03_create_gold_tables
04_run_data_quality_checks
```

7. Alternatively, execute:

```text
05_banking_governance_pipeline
```

8. Validate the expected output:

```text
Bronze tables:      7
Silver tables:      7
Gold tables:        3
Governance tables:  1
Total tables:      18
```

9. Connect Power BI to the Fabric Lakehouse.
10. Create the DQ and sales semantic models and reports.

---

## Validation Results

Expected controlled results:

| Validation | Expected result |
|---|---:|
| Golden legal entities | 98 |
| Source contract rows | 1,025 |
| Unique contracts after deduplication | 1,000 |
| Governed contracts | 724 |
| Excluded contracts | 276 |
| Governed contract share | 72.4% |
| Data quality rules | 12 |
| Uniqueness issue occurrences | 30 |

---

## Current Limitations

- The project uses synthetic data.
- Data quality issue ownership and remediation workflow are documented but not yet integrated with a ticketing system.
- Purview glossary, ownership, and lineage configuration are planned.
- The monthly KPI mart is dependent on the available reporting and target periods. Fact-table and KPI-mart totals should always be reconciled using the same date scope.
- Data quality failure totals represent rule failure occurrences rather than guaranteed unique affected records.

---

## Planned Improvements

- Issue management functionality
- Improve Fabric app with sales data
- React Three 3D data lineage
- Data domain and data product definitions
- Critical Data Element classification
- Automated lineage
- Data owner and steward assignments
- Automated alerts for critical DQ failures
- Historical DQ trend reporting
- Root-cause and remediation workflow
- CI/CD across development, test, and production workspaces
- Automated reconciliation tests between the sales fact and KPI mart
- Microsoft Purview business glossary

---

## Documentation

Detailed project documentation is maintained in the `/docs` folder:

- Business Requirements Specification
- Functional Specification
- Data Quality Rule Catalogue
- Data Dictionary
- Dashboard User Guide

---

## Author

**Allan Vissor**

Portfolio project focused on:

- data governance;
- data quality;
- analytics engineering;
- banking reporting;
- Microsoft Fabric;
- Power BI;
- business analysis.
