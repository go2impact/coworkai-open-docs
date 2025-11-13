# CoworkAI Cloud Operations & Medallion Architecture

Google Cloud reference showing how CoworkAI applies a Delta Lake-style medallion pattern (Bronze → Silver → Gold) while orchestrating ELT, AI enrichment, and delivery.

## 1. Reference Architecture (GCP)

```
                        +-----------------------------------------+
Telemetry Collectors    |  Google Cloud Pub/Sub (multi-topic bus) |
(activities, chat, etc.)+-------------------+---------------------+
                                         ingest events
                                          ▼
                                 +---------------+
                                 | Dataflow (EL) |
                                 |  Streaming    |
                                 +-------+-------+
                                         │ writes raw delta files
                                         ▼
                               +-----------------------+
                               | Bronze Delta Tables   |
                               | (GCS + Delta Lake)    |
                               +-----------------------+
                                         │ Auto-compaction via Dataproc Serverless + Delta Lake OPTIMIZE
                                         ▼
                     Cloud Composer DAG orchestrates ▼                           
+----------------------+         +------------------------+         +---------------------------+
| Dataproc Serverless  |  -----> | Silver Delta Tables    |  -----> | Gold Layer (BigQuery      |
|  (Batch notebooks    | enrich  | (normalized telemetry, |  refine |  marts + Looker Models)   |
|   & Delta jobs)      | stages  | manifests, jobs, KPIs) |  joins  +---------------------------+
+----------+-----------+         +-----------+------------+                 ▲            ▲
           │                                   │                            │            │
           │                                   │                            │            │
           │    +------------------------------+----------------------------+            │
           │    |                          Data Processing Stage 1 (Workblock builder)    |
           │    |  GCS «silver» >> ProductivyInsightsRunner on GKE Autopilot or Cloud Run |
           │    +------------------------------+----------------------------+            │
           │                                      │                                     │
           │                                      ▼                                     │
           │                           +---------------------------+                     │
           │                           | AI / Insight Services     |                     │
           │                           | (Vertex AI / WorkHorseLLM)|                     │
           │                           +-------------+-------------+                     │
           │                                         │                                   │
           ▼                                         ▼                                   │
+-----------------------+             +-------------------------------+                  │
| Delta Lake Gold Log   |             | GCS Output Root (markdown/    |------------------+
| (context & briefs)    |             | parquet/briefs)               |
+----------+------------+             +-------------------------------+
           │                                                │
           ▼                                                ▼
   BigQuery External Tables                          Looker Studio / Partner APIs
```

## 2. Component Roles

| Layer | GCP Services | Purpose |
| --- | --- | --- |
| Ingestion | Cloud Pub/Sub, Cloud Dataflow streaming | Buffer multimodal telemetry, apply lightweight PII tokenisation, land raw Delta files (Bronze) on GCS with exactly-once semantics. |
| Orchestration | Cloud Composer (Airflow) | Schedules ELT DAGs, backfills medallion layers, triggers ProductivityInsightsRunner workloads, manages dependency-aware SLAs. |
| Storage | GCS + Delta Lake | Delta format on object storage for Bronze/Silver layers, enabling ACID merges, time travel, and schema evolution before surfacing to BigQuery. |
| Transformation | Dataproc Serverless (Spark + Delta Lake), Cloud Dataflow batch, ProductivityInsightsRunner (GKE Autopilot or Cloud Run) | Spark jobs prep medallion layers; runner consumes Silver tables, performs Stage 1 processing, and writes structured outputs. |
| AI/Insights | Vertex AI endpoints + WorkHorseLLM (Gemini) | Execute Stage 2 summarisation, context consolidation, and CoworkAI brief generation with Upstash rate limiting. |
| Analytics & Serving | BigQuery, Looker, Partner APIs (Cloud Run), BigQuery Omni/External tables | Present Gold metrics/dashboards, power partner integrations. |
| Observability & Ops | Cloud Logging, Cloud Monitoring, Error Reporting, Cloud Build, Artifact Registry | Log processing outcomes, emit KPIs, manage CICD for containers and DAGs. |

## 3. Medallion Layers Mapped to CoworkAI

| Layer | Contents | Producer | Consumers |
| --- | --- | --- | --- |
| **Bronze (Raw)** | Unmodified telemetry, manifests, chat payloads, jobs CSV snapshots, device posture, clipboard events | Dataflow streaming jobs writing Delta tables | Dataproc Delta OPTIMIZE/COMPACT, forensic queries |
| **Silver (Refined)** | Normalised activity timelines, cleaned keystrokes, paired manifests with KPI focus areas, enriched org metadata, telemetry health audit tables | Dataproc Serverless Delta pipelines (invoked by Composer) | ProductivityInsightsRunner (Stage 1), anomaly dashboards, governance checks |
| **Gold (Curated)** | Workblock aggregates, atomic slices, insights, token contexts, CoworkAI briefs, KPI rollups; exposed as BigQuery marts + external Delta logs | ProductivityInsightsRunner + AI services writing to GCS + BigQuery load jobs | Customer dashboards, partner APIs, regulatory packs |

Delta Lake transactions ensure schema-on-write consistency across layers while supporting time-travel backfills when upstream data is reprocessed.

This cloud ops view complements the runtime architecture doc, demonstrating that CoworkAI fits cleanly into standard GCP ELT patterns while preserving Delta Lake lineage and enterprise governance expectations.
