# CoworkAI Open Docs

This repository collects reference documentation for the CoworkAI telemetry and insight pipelines. It is an extracted snapshot of the private CoworkAI repo documentation, curated for public consumption. Use the table of contents below to jump into the source files that explain how raw activity is transformed into AI-ready briefs.

```
                         +-----------------------+
Telemetry Sources        |  Governance & Secrets |  direnv / GCS creds / API keys
(activities, chat, etc.) +-----------+-----------+
            │                        │
            ▼                        ▼
+------------------+       +---------------------+
|  Data Landing &  |       |  Make/Poetry/CLI    |  reproducible runs
|  Manifest Intake |       |  (ProductivityInsightsRunner)
+--------+---------+       +----------+----------+
         │                            │ orchestrates
         ▼                            ▼
+--------+------------------------------------------------------------------+
|          Stage 0 – Shared Enrichment & Governance Hub                      |
| DataSourceManager                                                          |
|   ├─ ManifestProcessor → paired SOD/EOD + KPI focus areas                  |
|   ├─ ChatDataProcessor → parquet + markdown chat artefacts                 |
|   ├─ JobsProcessor → org/role snapshot for enrichment                      |
|   ├─ KeystrokeProcessor → density/quality metrics for telemetry gating     |
|   ├─ ActivityProcessor → cross-app normalization & session integrity       |
|   ├─ ClipboardProcessor → redaction + policy compliance checkpoints        |
|   ├─ SystemHardwareProcessor → device posture, idle signals                |
|   └─ Drift/Schema Monitors → failure ledger, observability hooks           |
+--------+------------------------------------------------------------------+
         │ batched artefacts (parquet/md)
         ▼
+--------+-----------------------------+------------------------------------+
|               Stage 1 – Data Processing Pipeline                           |
| Data processing modules → WorkblockBuilder → BatchResultsWriter            |
+--------+-----------------------------+------------------------------------+
         │ structured outputs & markdown reports
         ├───────────────────────────────────────┐
         ▼                                       │ reuse
+--------+-----------------------------+---------▼--------------------------+
|            Stage 2 – Insight & AI Services                                   |
| WorkHorseLLM | TokenBasedContextBuilder | TokenBasedBriefGenerator          |
+--------+-----------------------------+---------+--------------------------+
         │ consolidated contexts, CoworkAI briefs
         ▼
+--------+-----------------------------+------------------------------------+
|                 Stage 3 – Delivery & Access                                  |
| GCS Output Root (parquet, markdown, briefs, logs) → Analytics, Dashboards     |
+-------------------------------------------------------------------------------+
```

## Table of Contents
- [Technical “How It Works” Walkthrough](how_it_works_architecture.md) — architecture narrative with ASCII diagram, lifecycle table, component deep dives, and operational guarantees.
- [Cloud Ops & Medallion Architecture](cloud_ops_architecture.md) — GCP ELT reference with Pub/Sub → Dataflow → Delta Lake medallion tiers, Composer orchestration, and delivery through BigQuery/Looker.
- [System Architecture Overview](system_architecture_overview.md) — conceptual map of the platform components, processing stages, storage layout, and operational considerations for running the pipeline at scale.
- [Core Concepts & Terminology](CORE_CONCEPTS.md) — defines the end-to-end data hierarchy (activities → workblocks → atomic slices → briefs), core entities, and key processing rules such as overlap resolution and keystroke matching.
- [From Raw Data to Insights](how_insights_work_simple.md) — quick-start narrative of the four-step prompt flow from captured telemetry through workblock analysis, chat analysis, and final brief creation.
- [Data Artifacts Required for Final Insights Generation](data_artifacts.md) — inventories every artefact written during enrichment, workblock processing, AI enhancement, and brief generation, with paths, producers, and payload contents.
- [Atomic Slices Definition & Implementation Guide](ATOMIC_SLICES_DEFINITION.md) — explains how atomic slices are constructed, why minute-level keystroke aggregation sets hard boundaries, and how consolidation reduces token usage without losing telemetry.
- [4-Phase Keystroke Matching Algorithm Reference](4_phase_matching_algorithm_reference.md) — details the preprocessing and four matching phases that guarantee 100% keystroke attribution, including synthetic gap/orphan activities and cross-app detection.
