# Core Concepts & Terminology

This document defines the fundamental concepts used throughout the CoworkAI system. Understanding these terms is essential for working with the data pipeline and interpreting the insights generated.

## Data Hierarchy Overview

The system transforms raw activity data through progressive refinement:

```
Raw Activities & Keystrokes → Clean & Resolve Overlaps → Match via 4-Phase Algorithm → Workblocks → Atomic Slices → Insights → Consolidated Context → Brief
```

## Core Concepts

### Workblock

A **workblock** is a continuous work session where activities are grouped based on temporal proximity. It represents a natural unit of work that may span multiple applications and tasks.

**Key Characteristics:**
- Activities belong to the same workblock if the gap between them is **≤60 minutes**
- A new workblock starts when there's a gap **>1 hour** between activities
- Minimum duration: **5 minutes** (shorter workblocks are filtered as noise)
- Identified by deterministic IDs in format: `wb:<user_token>:<time_token>:<block_token>`
  - `user_token` is a 10-character base62 digest of the user identifier (~60 bits of entropy)
  - `time_token` is the workblock start timestamp (UTC) encoded in base62 milliseconds
  - `block_token` is a zero-padded base62 counter reflecting the workblock order for the user
  - Example: `wb:31my5iJtNx:U06GFvM:00`
- IDs are deterministic for a user’s data; recomputing the same day produces matching identifiers.
- Groups activities purely by time, not by application or task type

**Example:**
```
9:00-9:30  Email (Outlook)          ┐
9:45-10:30 Coding (VSCode)          ├─ Same workblock (gaps ≤1 hour)
10:45-11:15 Research (Chrome)       ┘

12:30-1:00 Email (Outlook)          ┐─ New workblock (gap >1 hour from previous)
```

**Note:** Workblocks with duration less than 5 minutes are automatically filtered out as noise.

### Atomic Slice

An **atomic slice** is the smallest meaningful unit of continuous work activity. It represents an uninterrupted period where the user stays in the same application and context (URL for browsers).

**Key Characteristics:**
- Same application AND same URL throughout the duration
- Bounded by context switches OR minute boundaries (when keystrokes present)
- Duration: Minimum 0.1 minutes (6 seconds) up to many minutes when consolidated
- Maximum 1 minute per slice when keystrokes are present (due to minute-level keystroke aggregation)
- Can be consolidated when no keystrokes exist (reduces token usage while preserving data)
- Part of a larger workblock
- Contains associated keystroke data
- Identified in storage by the tuple `(workblock_id, sequence)`; downstream consumers treat this pair as the primary key.

### Activity

An **activity** is the raw data representing which application was active during a specific time period. Activities are the foundation that gets processed into workblocks and atomic slices.

**Characteristics:**
- Records application name, window title, and time range
- May contain overlaps in the raw data (resolved by trimming earlier activity end times)
- Gets cleaned and normalized before workblock creation
- Application names are normalized to canonical forms (e.g., "chrome.exe" → "chrome")
- Minimum meaningful duration after cleaning: varies by source

### Keystroke Data

**Keystroke data** captures what was typed during work sessions, providing insights into productivity patterns.

**Characteristics:**
- Aggregated at the minute level (not individual keystroke timing)
- Includes timestamp, text typed, application info, and sometimes URL
- Matched to activities using the 4-phase algorithm
- Used to calculate efficiency metrics like error rate, shortcut usage, and copy-paste patterns
- Privacy-preserving: minute-level aggregation prevents detailed timing analysis

### Workblock Report

A **workblock report** is the human-friendly markdown document generated for each user/date run. It bundles the structured view of workblocks and their atomic slices together with enrichment data.

**Contains:**
- Workblock overview table with duration, active time, and atomic-slice count
- Per-slice timeline detailing application, URL, and keystroke excerpts
- Job and manifest context when available (e.g., area of focus, KPIs)
- Usage statistics tables (top applications, domains, URLs)

**Example Use Case:** "Review what I accomplished yesterday and share highlights with my manager."

### Usage Statistics

**Usage statistics** are aggregated rankings (top 10 each) derived from atomic slices to show where time was invested.

**Contains:**
- Applications by total duration and share of the day
- Websites/domains with time spent
- Exact URLs with duration and percentage

**Example Use Case:** "Identify which sites consumed most of my time during sprint planning."

### Consolidated Context Bundle

A **consolidated context** bundle is the token-bounded multi-day snapshot assembled by the token-based context builder. It is persisted under `context_consolidated/` and consists of multiple coordinated artefacts:

- `context_{start}_to_{end}.md` – Narrative markdown combining recent reports, insights, and chat summaries.
- `context_{start}_to_{end}.json` – Metadata capturing token counts, date range, and included parts.
- `context_component_log_{start}_to_{end}.json` – Detailed log of artefacts included, skipped, or missing.

**Example Use Case:** "Feed the latest three days of work into the brief generator without exceeding the token budget."

### Brief

A **brief** is the structured insight package created by the brief generator. Each brief is stored under `brief_consolidated/primary_date=.../` in both JSON and markdown formats and references the consolidated context range it summarizes.

**Contains:**
- Insight list with evidence-backed recommendations
- Momentum, blockers, and next-step narrative tailored to the individual contributor
- Machine-readable JSON payload for downstream integrations (includes `context_range`)

**Example Use Case:** "Deliver a concise update to leadership based on the latest consolidated context."

## Data Processing Philosophy

The system follows a "progressive data refinement" approach:

1. **Raw Data**: Unprocessed activity and keystroke logs
2. **Cleaned Data**: Normalized apps, resolved overlaps, privacy-filtered
3. **Matched Data**: Activities enriched with keystrokes via 4-phase algorithm
4. **Workblocks**: Temporally grouped meaningful work sessions and atomic slices
5. **Insights & Reports**: Human-readable markdown with enrichment context and usage statistics
6. **Consolidated Context**: Multi-day token-bounded bundle ready for AI summarisation
7. **Brief**: Final structured insight package linked to its source context

Each stage adds value while preserving privacy and reducing data volume.

### 4-Phase Keystroke Matching Algorithm

The system ensures 100% keystroke matching through a sophisticated 4-phase algorithm:

1. **Phase 1 - Direct Match**: Matches keystrokes to activities with exact temporal overlap AND same application
2. **Phase 2 - Nearest Neighbor**: For unmatched keystrokes, finds nearest activity within 5 minutes, preferring same-app matches (3-minute penalty for cross-app)
3. **Phase 3 - Gap Filling**: Creates synthetic activities for gaps between real activities where keystrokes exist
4. **Phase 4 - Orphan Activities**: Groups remaining keystrokes by app and temporal proximity into synthetic "orphan" activities

**Result**: Every keystroke is matched to an activity, ensuring no data loss.

### Overlap Resolution

When activities overlap in time (common in raw data), the system:
- Sorts activities chronologically
- Trims the end time of earlier activities to match the start time of the next
- Preserves all activities while eliminating overlaps
- Recalculates durations after resolution


## Relationships

```
Workblock (1) ─────┬─────> Atomic Slice (Many)
                   │              │
                   │              ├─> Contains Activities
                   │              └─> Contains Keystrokes
                   │
                   └─────> Workblock Report (per user/date)
                                  │
                                  ├─> Usage Statistics (aggregations)
                                  └─> Feeds Consolidated Context ──> Brief
```

## Usage Examples

### Understanding Your Work Patterns
- **Workblocks** reveal natural work sessions and gaps that became breaks.
- **Atomic slices** provide minute-level evidence with keystroke attribution.
- **Workblock reports** consolidate the day’s activity, enrichment, and usage stats.
- **Consolidated contexts** let you inspect which artefacts were retained or skipped when building AI-ready bundles.

### Improving Productivity
- Review **usage statistics** to see which applications or domains dominated the day.
- Compare consecutive **workblock reports** to spot shifts in focus or collaboration.
- Use **briefs** to communicate momentum, blockers, and next steps without rereading raw telemetry.
- Inspect the **context component log** when tuning token budgets or debugging missing evidence.

---

### Usage Statistics (Top 10 each)
- **Applications**: By total duration with percentage of day
- **Websites**: By domain with time spent
- **URLs**: Exact pages visited with duration

---

**Note**: Atomic slices represent the smallest unit of activity tracking, bounded by minute intervals when keystrokes are present or context switches occur.

**Important**: While atomic slices are bounded at 1-minute intervals when keystrokes are present (due to minute-level keystroke aggregation), consecutive slices with no keystrokes are consolidated to reduce token usage. This means you may see atomic slices lasting many minutes in the output when the user was reading or idle.
