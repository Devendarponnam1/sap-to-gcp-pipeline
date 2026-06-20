# SAP to GCP Data Migration Pipeline

A Python ETL pipeline that extracts data from an SAP source table, validates and cleanses it, maps SAP's native schema to an analytics-friendly BigQuery schema, and loads it into Google Cloud Platform with end-to-end integrity checks.

Built as a mentored, real-time project to practice the core skills used in enterprise SAP-to-cloud migrations: schema mapping, data validation, format conversion, and zero-data-loss load verification.

## Why this exists

Enterprises running SAP often need their operational data (material masters, financial postings, vendor records, etc.) available in a cloud data warehouse for analytics. This project demonstrates that migration pattern end-to-end, using SAP's MARA (Material Master) table as a representative example.

## Architecture

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐     ┌─────────────┐     ┌──────────────┐
│ SAP Source  │ --> │   Extract    │ --> │   Validate   │ --> │  Transform  │ --> │  Stage (GCS) │
│ (CSV/OData) │     │ sap_extractor│     │ data_validator│     │  schema_    │     │  + Load (BQ) │
└─────────────┘     └──────────────┘     └──────────────┘     │ transformer │     └──────────────┘
                                                 │              └─────────────┘
                                                 ▼
                                          rejected_rows.csv
                                          (every failure logged,
                                           nothing silently dropped)
```

**Pipeline stages:**

1. **Extract** (`src/extract/sap_extractor.py`) — Reads a flat-file SAP table export (the standard SE16N / scheduled-job extract pattern). An OData extraction mode is also implemented for SAP Gateway-based systems.
2. **Validate** (`src/validate/data_validator.py`) — Checks every row against schema rules (required fields, max length, date format, duplicate keys) driven entirely by config — no code changes needed to add a new rule. Rejected rows are written to `logs/rejected_rows.csv` with the exact reason, so failures are auditable rather than silently dropped.
3. **Transform** (`src/transform/schema_transformer.py`) — Renames SAP's short field codes (`MATNR`, `MTART`, ...) to clean column names, casts types, and converts SAP's native `YYYYMMDD` date format to proper `DATE` objects.
4. **Load** (`src/load/gcs_loader.py`, `src/load/bigquery_loader.py`) — Stages transformed data as Parquet to Cloud Storage, then loads into BigQuery with a post-load row-count reconciliation check.

## Schema mapping example

| SAP Field | SAP Meaning | BigQuery Field | Type |
|---|---|---|---|
| `MATNR` | Material Number | `material_number` | STRING |
| `MTART` | Material Type | `material_type` | STRING |
| `ERSDA` | Created On (YYYYMMDD) | `created_date` | DATE |
| `BRGEW` | Gross Weight | `gross_weight` | FLOAT64 |

Full mapping lives in [`config/schema_mapping.yaml`](config/schema_mapping.yaml) — adding a new SAP field to the pipeline is a config change, not a code change.

## Getting started

### 1. Install dependencies

```bash
pip install -r requirements.txt
```

### 2. Try it without any GCP setup (dry run)

The repo ships with sample SAP data (`sample_data/sap_mara_extract.csv`), including intentionally dirty rows (a duplicate key, a missing required field, an invalid date) so you can see validation working immediately:

```bash
python run_pipeline.py --dry-run
```

This runs extract → validate → transform and writes the result to `logs/dry_run_output.csv`, without touching GCP.

Example output:
```
============================================================
PIPELINE RUN SUMMARY
============================================================
  Rows extracted from SAP : 20
  Rows passed validation  : 17
  Rows rejected           : 3
  Rows transformed        : 17
  Loaded to BigQuery      : No (dry run)
============================================================
```

### 3. Run the full pipeline against real GCP

```bash
cp .env.example .env   # fill in your GCP project, bucket, and credentials path
export $(cat .env | xargs)
python run_pipeline.py
```

This stages the transformed data to Cloud Storage and loads it into BigQuery, verifying row counts on completion.

## Running tests

```bash
pip install pytest
pytest tests/ -v
```

Tests cover validation rules (required fields, duplicates, max length, date format) and transformation logic (renaming, type casting, date conversion) in isolation, without needing GCP credentials.

## Project structure

```
sap-to-gcp-pipeline/
├── config/
│   ├── config.yaml              # pipeline + connection settings
│   └── schema_mapping.yaml      # SAP field -> BigQuery field mapping
├── sample_data/
│   └── sap_mara_extract.csv     # sample SAP extract, includes dirty data
├── src/
│   ├── extract/sap_extractor.py
│   ├── validate/data_validator.py
│   ├── transform/schema_transformer.py
│   ├── load/gcs_loader.py
│   ├── load/bigquery_loader.py
│   └── utils/                   # config loader, logging
├── tests/
│   ├── test_data_validator.py
│   └── test_schema_transformer.py
├── run_pipeline.py              # entry point / orchestrator
├── requirements.txt
└── .env.example
```

## Design notes

- **Config-driven schema mapping** — extending the pipeline to a new SAP table or field is a YAML edit, not a code change.
- **Nothing is silently dropped** — every row that fails validation is written to `logs/rejected_rows.csv` with a specific rejection reason, which is what makes "zero data-loss" a checkable claim rather than an assumption.
- **Dry-run mode** — extract/validate/transform can be tested without any GCP credentials, which keeps local development fast and makes the pipeline easy to demo.
- **Type-safe extraction** — SAP IDs are read as strings (`dtype=str`) during extraction so leading zeros in material numbers and similar fields aren't silently stripped by pandas' numeric type inference.

## Tech stack

Python · pandas · PyYAML · Google Cloud Storage · Google BigQuery · pytest

## License

MIT — free to use as a reference or starting point for your own SAP-to-cloud migration work.
