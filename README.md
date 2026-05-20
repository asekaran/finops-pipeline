# Multi-Cloud Cost Reporting System

> Automated billing pipeline across AWS, GCP, and Azure — one dashboard, zero manual exports.

---

## The Problem

Every month, the team manually:

- Logged into 3 separate cloud billing portals
- Exported CSVs and cross-referenced account IDs
- Copy-pasted numbers into PowerPoint slides
- Spent **4–6 hours per person** on a task that adds no analytical value

This repository documents the fully automated replacement: a pipeline that ingests billing data from all three clouds, lands it in a central data lake, and surfaces it in a self-refreshing Power BI dashboard.

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Multi-Cloud Cost Pipeline                            │
│                                                                              │
│  ┌─────────────┐                                                             │
│  │    AWS      │──── CUR 2.0 ──▶ S3 ────────────┐                          │
│  └─────────────┘                                 │                          │
│                                                  ▼                          │
│  ┌─────────────┐              ┌──────────┐   ┌──────────────┐  ┌─────────┐ │
│  │    GCP      │── BigQuery ──▶   ADF    │──▶│  ADLS Gen2   │─▶│Power BI │ │
│  └─────────────┘    Export    │ Pipelines│   │  (Data Lake) │  │Dashboard│ │
│                               └──────────┘   └──────────────┘  └─────────┘ │
│  ┌─────────────┐                                 ▲                          │
│  │   Azure     │──── Cost Management Export ─────┘                          │
│  └─────────────┘                                                             │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Data Lake Container Structure

```
ADLS Gen2 Storage Account
│
├── raw-aws/
│   └── cur2/
│       └── YYYY/MM/
│           └── *.parquet          ← AWS Cost & Usage Report 2.0
│
├── raw-gcp/
│   └── billing_export/
│       └── YYYY/MM/
│           └── *.json / *.parquet ← GCP BigQuery billing export
│
└── raw-azure/
    └── cost-export/
        └── YYYY/MM/
            └── *.csv              ← Azure Cost Management export
```

---

## Stack

| Layer | Service | Purpose |
|---|---|---|
| **Ingestion — AWS** | AWS Cost & Usage Report 2.0 | Exports detailed billing data to S3 monthly |
| **Ingestion — GCP** | GCP BigQuery Billing Export | Streams billing data into BigQuery, piped out |
| **Ingestion — Azure** | Azure Cost Management Export | Scheduled monthly CSV export to ADLS |
| **Storage** | Azure Data Lake Storage Gen2 | Central lake with dedicated containers per cloud |
| **Orchestration** | Azure Data Factory | Pipelines to move and normalise data across sources |
| **Reporting** | Power BI (Service) | Dashboard with 13 months of history, weekly refresh |

---

## Data Flow — Step by Step

### 1. AWS → S3 (CUR 2.0)

AWS Cost & Usage Report 2.0 is configured to export billing data as Parquet files to an S3 bucket on a monthly schedule. CUR 2.0 uses a column-based format that is significantly more queryable than the legacy CSV format.

```
AWS Billing → CUR 2.0 Export → S3 Bucket (us-east-1)
                                    └── cost-reports/YYYY/MM/
```

Key configuration:
- Export type: `COST_AND_USAGE` with resource-level detail
- Format: Parquet (columnar, compressed)
- Cadence: Monthly, delivered within 24h of month close
- Granularity: Daily

### 2. GCP → BigQuery Export

GCP billing data is enabled at the organisation level and streams into a BigQuery dataset automatically. An ADF pipeline reads from BigQuery using the BigQuery connector and writes to `raw-gcp/` in ADLS.

```
GCP Billing API → BigQuery Dataset (billing_export)
                       └── ADF Pipeline → ADLS raw-gcp/
```

### 3. Azure → ADLS (Cost Management)

Azure Cost Management scheduled exports deliver a CSV to `raw-azure/` automatically each month — no pipeline step required. This is the simplest ingestion path.

```
Azure Cost Management → Scheduled Export → ADLS raw-azure/ (direct)
```

### 4. ADF Orchestration

Azure Data Factory hosts two active pipelines:

| Pipeline | Source | Destination | Trigger |
|---|---|---|---|
| `pl_ingest_aws_cur` | S3 (CUR 2.0 Parquet) | `raw-aws/` in ADLS | Monthly, day 3 |
| `pl_ingest_gcp_billing` | BigQuery export dataset | `raw-gcp/` in ADLS | Monthly, day 3 |

Azure billing lands directly into ADLS via Cost Management — no ADF pipeline needed for that path.

### 5. Power BI — Dashboard & Refresh

Power BI connects directly to ADLS Gen2 via the Azure Data Lake connector. The dataset is published to Power BI Service with a weekly scheduled refresh every Monday morning.

Current dashboard views:
- Monthly cost by cloud (stacked bar, 13 months)
- Month-over-month delta by service
- Top 10 cost drivers per cloud
- YTD spend vs same period last year

---

## What's Been Built

- [x] ADLS Gen2 storage account with `raw-aws`, `raw-gcp`, `raw-azure` containers
- [x] Azure Cost Management export — zero-touch monthly delivery
- [x] AWS CUR 2.0 — Parquet export to S3 configured and live
- [x] ADF pipeline for AWS S3 → ADLS
- [x] Power BI report with 13 months of Azure billing history
- [x] Power BI Service publish with weekly auto-refresh

## What's In Progress

- [ ] ADF pipeline for GCP BigQuery → ADLS
- [ ] Combined 3-cloud unified Power BI dataset
- [ ] Cross-cloud cost normalisation (unified currency, tag taxonomy)
- [ ] Cost spike alerts via Teams (Power Automate + Power BI Data Alerts)

---

## Time Savings

| Task | Before | After |
|---|---|---|
| Pull AWS billing data | ~10 min (manual export, format CSV) | 0 min (automatic) |
| Pull GCP billing data | ~15 min (BigQuery console, manual export) | 0 min (automatic) |
| Pull Azure billing data | ~10 min (portal export) | 0 min (automatic) |
| Build monthly slide deck | ~20 min (copy-paste, chart rebuild) | 0 min (dashboard self-updates) |
| **Total per month** | **~1 hour** | **~0 hours** |

---

## Cost Considerations

This pipeline runs on Azure. Approximate monthly infrastructure cost at low data volumes (< 10 GB/month across all three clouds):

| Service | Estimated Cost |
|---|---|
| ADLS Gen2 storage (< 10 GB) | ~$0.02 |
| ADF pipeline runs (2 pipelines × monthly) | ~$1–2 |
| Power BI Pro licence (per user) | $10/user/month |
| AWS S3 storage (CUR export, < 1 GB) | ~$0.02 |

Total infrastructure cost is negligible. Power BI Pro is the only meaningful ongoing cost if not already licensed.

---

## Design Decisions

**Why ADLS Gen2 as the central lake?**
All three clouds can write to Azure Blob-compatible storage. ADLS Gen2 adds hierarchical namespace (directory-level operations, fine-grained ACLs) while remaining compatible with the Blob API — so AWS can write directly via the Azure Blob SDK, and GCP can write via ADF.

**Why keep raw containers per cloud?**
Billing schemas differ significantly across AWS, GCP, and Azure. Keeping raw data isolated by source makes debugging easier and allows independent schema evolution. A separate transformation layer (not yet built) will normalise into a unified schema for the combined dashboard.

**Why Power BI and not a custom dashboard?**
The team already uses Power BI. The priority was eliminating manual work, not introducing new tooling. Power BI's direct ADLS connector and scheduled refresh solve the problem with zero custom code.

**Why not Microsoft Fabric?**
Fabric would consolidate ADLS, ADF, and Power BI into a single platform. It's a viable future migration path — but adopting Fabric would mean re-platforming three working services simultaneously. The current stack is well understood, cost-effective, and fully functional.

---

## Repo Structure

```
multi-cloud-cost-reporting/
│
├── adf/
│   ├── pipelines/
│   │   ├── pl_ingest_aws_cur.json
│   │   └── pl_ingest_gcp_billing.json
│   ├── datasets/
│   │   ├── ds_s3_cur_parquet.json
│   │   ├── ds_bigquery_billing.json
│   │   └── ds_adls_raw.json
│   └── linked_services/
│       ├── ls_aws_s3.json
│       ├── ls_gcp_bigquery.json
│       └── ls_adls_gen2.json
│
├── powerbi/
│   └── MultiCloudCosts.pbix
│
├── docs/
│   ├── aws-cur-setup.md
│   ├── gcp-bigquery-export-setup.md
│   └── azure-cost-management-setup.md
│
└── README.md
```

---

## Setup Guides

### AWS — Enable CUR 2.0

1. Go to **AWS Billing → Data Exports → Create Export**
2. Select export type: `Cost and Usage Report`
3. Set delivery location to your S3 bucket
4. Set format to **Parquet**, granularity **Daily**, compression **Snappy**
5. Enable resource IDs and split cost allocation data

Full guide: [`docs/aws-cur-setup.md`](docs/aws-cur-setup.md)

### GCP — Enable BigQuery Billing Export

1. In Google Cloud Console go to **Billing → Billing Export**
2. Enable **Standard usage cost** export to BigQuery
3. Select or create a BigQuery dataset (e.g. `billing_export`)
4. Create an ADF Linked Service pointing to the dataset using a GCP service account

Full guide: [`docs/gcp-bigquery-export-setup.md`](docs/gcp-bigquery-export-setup.md)

### Azure — Enable Cost Management Export

1. Go to **Cost Management → Exports → Add**
2. Export type: **Monthly cost by resource**
3. Storage: point to your ADLS Gen2 account, container `raw-azure`
4. Schedule: run on day 3 of each month (allows billing to settle)

Full guide: [`docs/azure-cost-management-setup.md`](docs/azure-cost-management-setup.md)

---

## Roadmap

```
Phase 1 — Foundation (complete)
  ✓ ADLS Gen2 data lake
  ✓ Azure billing auto-export
  ✓ AWS CUR 2.0 → S3
  ✓ Power BI with Azure data, weekly refresh

Phase 2 — Full Coverage (in progress)
  ◻ GCP BigQuery → ADLS pipeline
  ◻ AWS S3 → Power BI connection
  ◻ Unified 3-cloud dataset

Phase 3 — Intelligence (planned)
  ◻ Cross-cloud cost normalisation layer
  ◻ Cost spike alerts via Teams
  ◻ Chargeback / showback by team tag
  ◻ Anomaly detection on spend trends
```

---

## Tags

`#Azure` `#AWS` `#GCP` `#FinOps` `#DataEngineering` `#PowerBI` `#MultiCloud` `#CloudCost` `#ADF` `#DataLake` `#Automation` `#BuildInPublic`
