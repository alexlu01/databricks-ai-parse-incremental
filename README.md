# Incremental Claims Parsing on Databricks (AI Parse + SDP)

A starter **Lakeflow Spark Declarative Pipeline (SDP)** that turns a backlog of scanned PDFs into
structured, searchable data using Databricks AI Functions. Built for high-volume document
intelligence (for example, OCR over a large claims archive), processed **incrementally** so each
document is parsed exactly once and re-runs only pick up new files.

## What it does

The pipeline (`notebooks/ai_parse_incremental.ipynb`) declares four streaming tables:

| Table | Layer | What happens |
|-------|-------|--------------|
| `claims_raw` | Bronze | Auto Loader ingests raw PDFs/images incrementally from a Volume |
| `claims_parsed_bronze` | Bronze | `ai_parse_document` (v2.0) runs OCR + layout, once per document |
| `claims_elements_silver` | Silver | Parsed output exploded to one row per page element |
| `claims_fields_silver` | Silver | `ai_extract` (v2) pulls example claim fields |

SDP manages checkpoints, retries, and incremental processing for you. There is no manual
`writeStream` or checkpoint path.

## Architecture

See [`docs/architecture/index.html`](docs/architecture/index.html) for the full reference architecture diagram (self-contained, opens in any browser).

## Clone it into your Databricks workspace (Git folder)

1. In your workspace, go to **Workspace → (your home) → Create → Git folder**.
2. Paste this repository's URL and clone it. The notebook appears as a Git folder you can pull
   updates into later.

## Run it as a pipeline

1. Open `notebooks/ai_parse_incremental.ipynb` and set `SOURCE_VOLUME_PATH` to the Unity Catalog
   Volume holding your documents.
2. Go to **Jobs & Pipelines → Create → Lakeflow Pipeline**, add this notebook as the source,
   set a **target catalog and schema** for the output tables, and choose **Serverless**.
3. Click **Start**. Re-running only processes new files.

## Prerequisites

- **AI Functions enabled** on the workspace (`ai_parse_document`, `ai_extract`, part of Agent
  Bricks). Confirm approval for sensitive data with your platform/security team.
- **DBR 15.4 LTS or above**, in a Foundation Model APIs supported region. The notebook sets the
  `preview` channel on the AI-function tables, which is required to use AI functions in a pipeline.
- A **Unity Catalog Volume** with a sample of documents to start (include some hard scans: old,
  faxed, handwritten).

## Notes for scaling up

- Throughput scales with **pages, not files**. A few very long documents can dominate a run.
- For documents over 500 pages or 100 MB, parse in ranges with the `pageRange` option.
- Documents that fail (often encrypted PDFs) surface via `parsed:error_status`; re-render those to
  images and re-run, or handle them in a small side flow.
- At sustained scale, request a model-unit / provisioned-throughput increase if you see throttling.
