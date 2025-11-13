# System Architecture Overview

This document describes the end-to-end behaviour of the platform at a conceptual level. It explains how telemetry flows through ingestion, enrichment, analytics, and optional AI summarisation to produce artefacts that power downstream experiences.

---

## 1. Architectural Summary

```
Telemetry Sources (activities, keystrokes, chat, manifests, jobs)
        │
        ▼
Data Source Preparation (enrichment pipelines)
        │
        ▼
Per-User Processing (daily workblock pipeline)
        │
        ├──► Structured Outputs (parquet tables, reports)
        │
        └──► Optional AI Layer (insights, context bundles, briefs)
```

At a high level the system runs in two phases:

- **Daily batch processing** ingests telemetry for the requested date, normalises it into workblocks, atomic slices, manifests, and job enrichment, then persists both analytical parquet tables and markdown reports under the structured `workblock_processing/` and companion directories.
- **On-demand AI summarisation** revisits the artefacts already written for a user, consolidates as many recent days as token limits allow, and emits enhanced narratives plus briefs back into the output tree.

All artefacts are written under a single object-storage output root using consistent partitioning (date and user identifiers) so teams can integrate without bespoke path logic.

---

## 2. Component Interaction Overview

```
PipelineOrchestrator
├── DataSourcePreparer
│   ├── ManifestIngestion
│   ├── ChatHistoryIngestion
│   └── JobsEnrichmentLoader
└── Daily Processing Loop
    ├── Activity & Keystroke Cleaners
    ├── Workblock builder (usage statistics + markdown formatter)
    └── BatchWriter (parquet + reports)

Optional AI Layer
├── InsightsLLM (enhanced narratives)
├── TokenContextBuilder (multi-day context bundles)
└── BriefGenerator (final briefs)
```

- **Pipeline orchestrator** coordinates each run, preparing shared data and invoking the per-user pipeline.
- **Data source preparer** materialises auxiliary datasets so downstream steps can attach context without rereading raw sources.
- **Daily processing loop** handles user-level cleaning, enrichment, and batch persistence for workblocks, atomic slices, manifests, and reports.
- **Optional AI layer** consumes the stored artefacts to produce narratives, consolidated contexts, and briefs without repeating the ingestion work.

---

## 3. Data Inputs and Outputs

### Inputs
- **User activities & keystrokes** – time-series telemetry emitted by client collectors.
- **Chat conversations** – structured logs of workplace messaging.
- **Manifests** – start-of-day and end-of-day declarations that provide context about user intent.
- **Jobs enrichment** – organisational metadata (role, team, region) used to personalise insights.

### Outputs
- **Workblock processing** – partitioned parquet tables and markdown reports produced for each date and user batch.
- **Chat history** – normalised parquet tables and human-readable summaries aligned to the same partitioning strategy.
- **Jobs enrichment parquet** – optional `jobs.parquet` files stored alongside workblock batches so enrichment data can be re-used without rereading input sources.
- **Insights directory** – optional AI-enhanced narratives when the insights LLM is invoked.
- **Context consolidated** – token-bounded bundles (markdown narrative, metadata JSON, component log) that merge recent artefacts for multi-day reasoning.
- **Brief consolidated** – brief packages derived from the consolidated context.

All locations inherit the configured output root and follow a hive-style layout (`date=YYYY-MM-DD`, `user_<id>`, `primary_date=<date>`), simplifying cleanup, audit, and exploratory analysis.

---

## 4. Processing Stages

### Stage A – Data Source Preparation

Before user-level workloads begin, the platform inspects shared data sources and materialises any auxiliary datasets required for the run. This stage:

1. Detects whether manifests and chat logs are available for the target logical date.
2. Normalises and stores paired manifest records plus chat history in the output tree.
3. Loads static enrichment sets (such as job metadata) so later stages can attach organisational context without repeated I/O.

Failures are captured and surfaced alongside the main processing report so operators know which auxiliary feeds were unavailable.

### Stage B – Per-User Daily Pipeline

Once shared resources are ready, the system iterates over each user present in the source telemetry for the requested date. For every user the pipeline:

1. Cleans activity and keystroke series, resolving overlaps and aligning timestamps.
2. Derives workblocks and atomic slices, calculating usage metrics and aggregations.
3. Merges enrichment data (manifests, job records, usage statistics) to enrich the analytics output.
4. Writes batched parquet datasets plus date- and user-scoped markdown reports into their canonical locations under `workblock_processing/`.

Batch writing reduces the number of object store writes from thousands per run to tens, enabling large cohorts to finish quickly. Sequential processing keeps memory usage bounded and simplifies error handling—if a user fails, the run continues while recording the failure for later triage.

### Stage C – Optional AI Layer

When teams opt into AI-generated summaries, the platform provides two extra capabilities:

1. **Insight enhancement** – The insights LLM transforms base reports into narrative summaries tailored to end users and writes the enhanced markdown into the `insights/` directory next to the originating data.
2. **Token-based consolidation and brief generation** – The token context builder harvests a user’s historical artefacts (newest-first), assembles a multi-day context that fits within a configurable token budget, and stores the combined bundle under `context_consolidated/` as a markdown narrative plus structured metadata (`context_{start}_to_{end}.json`) and a component log (`context_component_log_{start}_to_{end}.json`). The brief generator reads the persisted context bundle and uses it as the sole input when converting the session into structured briefs saved beneath `brief_consolidated/`.

These steps are decoupled from the daily batch, allowing operators to rerun summarisation without reprocessing raw telemetry.

---

