# Data Artifacts Required for Final Insights Generation

This guide documents every artefact produced by the open pipeline and the location where it is written. All paths below are expressed relative to `<output_root>`, the configurable root directory (or bucket prefix) that teams choose when deploying the workflow.

## Stage 0 – Enrichment Data Processing

These preparatory datasets are materialised first so later stages can reuse them without re-reading source systems.

1. **Chat History**
   - **Location**: `<output_root>/chat_history/date={date}/batch_{batch}/chat_history.parquet` (batch parquet) and `<output_root>/chat_history/date={date}/user_{user_id}/chat_history_{date}.md` (per-user markdown).
   - **Produced by**: Chat history ingestion job.
   - **Contents**: Cleaned user-only conversations with timestamps and lightweight metadata. The markdown variant feeds the insights prompts.

2. **Paired Manifests**
   - **Location**: `<output_root>/manifests/paired/date={date}/manifests_paired_{date}.parquet`.
   - **Produced by**: Manifest pairing job.
   - **Contents**: Start-of-day and end-of-day intent records with goals, reflections, and KPI notes for each participant.

3. **Jobs Data**
   - **Location**: `<output_root>/workblock_processing/date={date}/batch_{batch}/jobs.parquet`.
   - **Produced by**: Organisational enrichment loader.
   - **Contents**: Role, team, manager, and other org metadata staged so the main pipeline can join it on demand.

## Stage 1 – Main Pipeline Processing

Core workblock construction, normalisation, and reporting.

4. **Workblocks Data**
   - **Location**: `<output_root>/workblock_processing/date={date}/batch_{batch}/`.
   - **Files**: `workblocks.parquet`, `atomic_slices.parquet`, optional `manifests.parquet`, and `jobs.parquet` copies for the batch.
   - **Produced by**: Workblock processing engine.
   - **Contents**: Session-level summaries plus atomic slice records with app, URL, keystrokes, and timing information.

5. **Workblock Report**
   - **Location**: `<output_root>/workblock_processing/date={date}/batch_{batch}/user_{user_id}/workblock_report_{date}.md`.
   - **Produced by**: Markdown formatter that renders the normalised workblocks and enrichment payloads.
   - **Contents**: Human-readable timeline with usage statistics, job context, and manifest excerpts.

## Stage 2 – AI Enhancement

Optional LLM passes that turn deterministic telemetry into narrative insights.

6. **Enhanced Workblock Insights**
   - **Location**: `<output_root>/insights/date={date}/user_{user_id}/workblock_insights_{date}.md`.
   - **Produced by**: Workblock insights generator.
   - **Inputs**: `workblock_report_{date}.md`.
   - **Contents**: Narrative highlighting focus periods, context switching, and productivity patterns.

7. **Enhanced Chat Insights**
   - **Location**: `<output_root>/insights/date={date}/user_{user_id}/chat_insights_{date}.md`.
   - **Produced by**: Chat insights generator.
   - **Inputs**: `chat_history_{date}.md`.
   - **Contents**: Themes from user prompts, collaboration signals, and prompt-engineering habits.

## Stage 3 – Final Insights Generation

Multi-day consolidation and brief creation. Teams can opt in when they need summarised narratives.

- **Consolidated Context Bundles**
  - **Location**: `<output_root>/context_consolidated/user_{user_id}/primary_date=YYYY-MM-DD/`.
  - **Files**: `context_{start}_to_{end}.md`, `context_{start}_to_{end}.json`, `context_component_log_{start}_to_{end}.json`.
  - **Produced by**: Token-based context builder.
  - **Contents**: Token-bounded markdown narrative, structured metadata (including token budget, date range, component counts), and a component log for auditability.

- **Final Briefs**
  - **Location**: `<output_root>/brief_consolidated/user_{user_id}/primary_date=YYYY-MM-DD/`.
  - **Files**: `brief_{start}_to_{end}.json` and `brief_{start}_to_{end}.md`.
  - **Produced by**: Brief generator leveraging the consolidated context.
  - **Contents**: Structured JSON for downstream integrations plus a human-readable summary. Metadata preserves the originating context range so briefs stay linked to their evidence.

## Dependency Overview

```
Raw Input Data
    ↓
[Stage 0: Enrichment]
    ├── Chat history ingestion → chat_history.parquet / chat_history.md
    ├── Manifest pairing → manifests_paired.parquet
    └── Jobs loader → jobs.parquet (batch copies)
    ↓
[Stage 1: Main Pipeline]
    ├── Workblock processor → workblocks.parquet, atomic_slices.parquet
    └── Markdown formatter → workblock_report.md (with jobs + manifests)
    ↓
[Stage 2: AI Enhancement]
    ├── Workblock insights generator → workblock_insights.md
    └── Chat insights generator → chat_insights.md
    ↓
[Stage 3: Final Insights]
    └── Brief pipeline
            ← context_{start}_to_{end}.md (+ metadata + component log)
            → brief_{start}_to_{end}.json + brief_{start}_to_{end}.md
```
