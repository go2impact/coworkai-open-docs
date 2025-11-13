# CoworkAI — Technical "How It Works"

## 1. Architecture at a Glance

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

### Explanation Highlights
- **Make/Poetry/CLI layer** guarantees identical execution whether the run is local or scheduled, exposing a single entry point (`ProductivityInsightsRunner`) and audited Make targets.
- **Shared enrichment (Stage 0)** auto-detects sources, validates schemas, pairs SOD/EOD manifests (including KPI/focus extraction), snapshots jobs data, batches chat markdown/parquet, and runs keystroke/activity/clipboard/system-hardware processors under a single governance hub before any per-user work begins.
- **Data processing pipeline (Stage 1)** runs deterministic cleaning → workblock construction → batched persistence, keeping memory bounded and producing both machine-ready (parquet) and human-ready (markdown) artefacts under hive-style partitions.
- **Insight & AI services (Stage 2)** consume the persisted artefacts, never raw telemetry, enabling multi-day context assembly and CoworkAI brief generation while preserving lineage.
- **Delivery layer (Stage 3)** centralises every artefact under a single GCS root with uniform partitioning (`date=`, `user_`, `primary_date=`), simplifying downstream analytics, governance, and retention policies.

## 2. End-to-End Flow

| Phase | What Happens | Key Outputs | Controls & Signals |
| --- | --- | --- | --- |
| Environment Bootstrap | `direnv` loads `.env`, `make auth-gcs` wires ADC, Poetry ensures pinned deps | Reproducible runtime, secrets isolated from repo | ADC checks, Make target logs |
| Stage 0 – Shared Enrichment | DataSourceManager orchestrates manifest pairing (KPIs, focus areas), chat batching, jobs snapshots, keystroke/activity/clipboard/system hardware vetting, schema/drift checks, and failure ledger updates | `chat_history/`, `manifests/paired/`, `jobs.parquet`, telemetry health logs | Schema validation, drift alerts, missing-source reporting, observability hooks |
| Stage 1 – Data Processing Pipeline | Clean per-user telemetry, build workblocks + atomic slices, merge enrichment, emit markdown reports | `workblock_processing/` parquet + `workblock_report.md` per user | Deterministic ordering, batch writers, failure ledger per user |
| Stage 2 – Insights & AI | WorkHorseLLM enhances reports, ContextBuilder assembles multi-day packets, BriefGenerator emits CoworkAI packages | `insights/`, `context_consolidated/`, `brief_consolidated/` | Token budgets, audit logs, optional live/offline modes |
| Stage 3 – Delivery | Artefacts exposed to BI tools, partner APIs, and downstream consumers | Date/user partitioned trees, optional dashboards | IAM-scoped GCS prefixes, lifecycle rules, checksum verification |

## 3. Component Deep Dive

### Orchestration Layer
- **ProductivityInsightsRunner** is the only entry point invoked by Make targets (`make run-demo`, `make test-brief-e2e`). It sequences Stage 0 → Stage 1 → Stage 2, records run metadata, and exposes toggles for AI workloads or report generation.
- **Automation Stack** uses Poetry for dependency locking, direnv for deterministic env injection, and Make for developer + CI parity. Maturity is evident in `Makefile` targets and `docs/external_tests*.md`.

### Shared Enrichment (Stage 0)
- **DataSourceManager** is the control tower: it auto-detects which feeds exist in the configured `INPUT_ROOT`, sequences processors, and records failures for operator review.
- **ManifestProcessor** pairs SOD/EOD manifests for the target date, extracts KPIs/focus areas, and persists canonical parquet that downstream joins reference uniformly.
- **ChatDataProcessor** batches chat telemetry into parquet for analytics plus markdown for LLM usage, guaranteeing the same partitioning layout (`chat_history/date=.../batch_...`).
- **JobsProcessor** snapshots organisational metadata so role/team context is available without rereading raw CSV/Parquet per user.
- **Telemetry Health Processors** (Keystroke, Activity, Clipboard, SystemHardware) validate signal density, reconcile cross-app sessions, enforce redaction policies, and capture device posture to gate downstream workloads.
- **Governance Hooks** enforce schema validation, drift detection, and a failure ledger; gaps are surfaced to Stage 1 so the run can proceed while flagging missing enrichments.

### Data Processing Pipeline (Stage 1)
- **Data processing modules** reconcile overlaps, timezone drift, and device quirks before analytics begin.
- **WorkblockBuilder** turns the cleaned timeline into semantic blocks plus atomic slices, layering on usage statistics.
- **BatchResultsWriter** flushes parquet batches (workblocks, atomic slices, manifests, jobs) and markdown reports to GCS with consistent partition names. Failed users are logged without halting the batch, which is key for large cohorts.

### Insight & AI Layer (Stage 2)
- **WorkHorseLLM** ingests the stored markdown reports and chat summaries to produce enhanced narratives, tagging each output with source paths for traceability.
- **TokenBasedContextBuilder** assembles multi-day contexts by scanning existing artefacts (newest-first) until a configurable token budget is reached; it stores markdown + JSON metadata + component logs.
- **TokenBasedBriefGenerator** consumes the consolidated context to build CoworkAI briefs in both markdown and JSON, enabling downstream APIs to stay structured.
- **Rate Limiting & Observability** rely on Upstash-backed permit acquisition plus structured logging of each LLM call (operation name, attempt counts, token usage).

### Delivery & Access (Stage 3)
- **Single Output Root** keeps every artefact under the configured `PRODUCTIVITY_INSIGHTS_OUTPUT_ROOT`, simplifying IAM and lifecycle policies.
- **Partitioning**: `date=YYYY-MM-DD` for batch artefacts, `user_<id>` for person-level markdown/insights, `primary_date=` on consolidated contexts/briefs so multi-day windows remain queryable.
- **Consumption Modes**: parquet tables feed BI or ML stacks, markdown/briefs power exec readouts, and JSON payloads back APIs or partner deliverables.

## 4. Operational Guarantees

- **Deterministic Runs** – Given the same telemetry slice, the pipeline produces identical parquet/markdown outputs; LLM stages record prompt + model metadata for audit.
- **Graceful Degradation** – Missing manifests or chat simply mark enrichment gaps while still emitting the rest of the data, preventing day-long outages.
- **Testing Discipline** – Make targets cover unit (`make test`), smoke (`make test-brief-smoke`), integration (`make test-brief-e2e`), ensuring each layer is verifiable prior to release.
- **Security & Compliance Hooks** – Secrets never live in-repo; `GOOGLE_APPLICATION_CREDENTIALS` and prompt stores stay external. Partitioned storage supports tenant isolation and audit logging.
- **Extensibility** – New signals plug in at Stage 0 via additional processors; new insights reuse Stage 1 artefacts, preserving lineage and reducing incremental risk.

## 5. Suggested Next Artifacts (Optional)

- **Mermaid/PNG Diagram** derived from the ASCII above for slide decks.
- **Synthetic Run Walkthrough** referencing `make run-demo` outputs (scrubbed) to give readers artefact samples.
- **Security Appendix** detailing IAM roles, audit log retention, and SOC2 roadmap checkpoints.

This draft stays focused on architecture and flow while remaining safe to share with external technical stakeholders.
