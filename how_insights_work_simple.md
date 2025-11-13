# From Raw Data to Insights

This quick guide explains how raw workplace signals are transformed into the brief. The process moves through four steps that blend deterministic data processing with targeted AI prompts.

## Step 1. Capture Workplace Signals

We collect several categories of telemetry before any prompts run:

- **Application activity** – which apps and websites were active, when they were used, and how long each session lasted.
- **Keystrokes** – minute-level aggregates of the keys pressed inside each app, preserving the original characters such as `<Cmd>`, `<Tab>`, and `<Backspace>`.
- **AI chat history** – the user’s questions to in-house copilots such as ChatGPT and Claude (responses from the AI are not ingested).
- **Goals and manifests** – daily plans, focus areas, and end-of-day reflections that provide intent and self-reported outcomes.

The main pipeline cleans these inputs, aligns timestamps, and builds structured workblocks that are ready for analysis. Prompt workflows reuse these normalised artefacts and never touch the raw feeds directly.

## Step 2. Workblock Analysis (workblock insights prompt)

- **Input**: `workblock_report_{date}.md`, which combines app usage, timing, keystrokes, and goal alignment for a single user and day.
- **Processor**: the workblock insights prompt analyses focus, identifies behavioural patterns, and groups related activities into case sessions while preserving the original telemetry.
- **Output**: `workblock_insights_{date}.md`, an enhanced narrative that calls out concentration periods, context switching, and noteworthy behaviours.

## Step 3. Chat Analysis (chat narrative prompt)

- **Input**: Sanitised AI chat transcripts (`chat_history_{date}.md`) that contain only the user’s prompts.
- **Processor**: the chat narrative prompt extracts themes, tracks project momentum, evaluates prompt engineering habits, and highlights blockers.
- **Output**: `chat_insights_{date}.md`, a readout of how the user leveraged copilots during the day.

## Step 4. Final Insights (final brief prompt)

- **Input**: The workblock and chat narratives plus structured metadata (goals, job context, manifests).
- **Processor**: the final brief prompt synthesises the combined signal, computes productivity scores, and validates the result against a strict JSON schema.
- **Output**: paired files `brief_{date}.json` and `brief_{date}.md`, which carry the structured brief and a human-readable summary.

## Prompt Flow Summary

1. The workblock insights and chat narrative prompts run in parallel, each producing a markdown insight file.
2. The final brief prompt ingests those narratives alongside the structured telemetry to produce the brief artefacts.

The deterministic pipeline guarantees complete, audit-ready telemetry, while the prompts layer turns the curated data into prioritised recommendations.
