# Atomic Slices Definition & Implementation Guide

## Executive Summary

Atomic slices are the fundamental unit of telemetry in our productivity insights system. Each slice represents user activity bounded by either context switches (app/window/URL changes) or minute boundaries.

**Why 1-minute maximum for slices with keystrokes?** Our keystroke data comes in 1-minute aggregations (e.g., all keystrokes from 09:00:00-09:00:59 are bundled together). We cannot split these keystrokes into smaller time windows. Therefore, we must create slice boundaries at every minute to ensure proper keystroke attribution. This constraint means:
- Maximum slice duration WITH keystrokes: 1 minute
- Consolidated idle slices (no keystrokes): Can span many minutes
- Minimum slice duration: 0.1 minutes (≈6 seconds) enforced during slice finalization
- Every row in our table = one atomic slice

## Core Definition

**From our Product Manager:**
> "The smallest, immutable telemetry segment: consecutive actions inside a single app/window/URL. Typical duration: Fractions of a second up to a few minutes, bounded by either a window/tab change or detectable inactivity."

### What is an Atomic Slice?

An **atomic slice** is a period of user activity bounded by:
- Context switches (app/window/URL changes) OR  
- Minute boundaries (required due to keystroke aggregation)

**The 1-Minute Constraint**: Since keystrokes are aggregated per minute (e.g., "09:00" contains all keystrokes from 09:00:00-09:00:59), we cannot attribute keystrokes to time periods longer than 1 minute. This forces us to create atomic slices at minute boundaries when keystrokes are present.

**Consolidation Optimization**: To reduce token usage and improve readability, consecutive atomic slices are consolidated when they have:
- Same application and URL context
- No keystrokes in any of the slices being consolidated
- Continuous time within the same context (bounded by context switches)

This optimization preserves minute boundaries when keystrokes exist (maintaining attribution accuracy) while reducing redundant slices during idle/reading periods. The consolidation happens within the same app/URL context, ensuring context switches are still respected.

### Key Characteristics

1. **Immutable**: Once created, never modified (preserve raw telemetry)
2. **Bounded by**: Context changes OR minute boundaries (when keystrokes present)
3. **Duration**: Minimum 0.1 minutes (≈6 seconds); maximum 1 minute for slices with keystrokes; consolidated idle slices can span many minutes
4. **Complete timeline**: All activity preserved, consolidated for efficiency
5. **Keystroke attribution**: Keystrokes distributed to appropriate minute slices
6. **Smart consolidation**: Idle periods merged, active periods preserved

## Data Constraints & Limitations

### What We Have

1. **Activity Data** (second precision)
   - Application name/executable
   - Window title and/or URL
   - Start and end timestamps
   - User ID

2. **Keystroke Data** (minute aggregation)
   - Timestamp (minute boundary, e.g., 09:15:00)
   - Actual keystrokes as text (all keystrokes from that minute bundled together)
   - Application context (may not match activity data)
   - No sub-minute timing
   - **Critical limitation**: Cannot split a minute's keystrokes across multiple slices

### What We DON'T Have (Current Limitations)

1. **Individual keystroke timestamps** - Cannot tell if typing happened at :05 or :55 within a minute
2. **Mouse/click data** - No visibility into mouse-driven work
3. **Scroll/navigation events** - Missing non-keyboard interactions  
4. **Sub-minute keystroke distribution** - Cannot spread keystrokes accurately within a minute
5. **True inactivity detection** - Cannot distinguish reading/thinking from idle time

**Note**: Future telemetry improvements will capture mouse events, scroll data, and finer-grained keystroke timing to address these limitations.

### The Core Challenge: Attribution Ambiguity

When multiple context switches occur within one minute, we cannot definitively attribute the keystrokes to specific contexts. Each keystroke in that minute is tagged with an `ambiguous` flag in the atomic slice payload, and the formatter uses that metadata to surface an asterisk (*) in rendered tables.

### Implementation Requirements

1. **Preserve raw data**: Never modify keystroke text or timestamps
2. **Sort stability**: Ensure consistent ordering when timestamps are identical
3. **Memory efficiency**: Process in streaming fashion for large datasets
4. **Validation**: Ensure no keystrokes are lost or duplicated

### Visual Representation

```
ACTIVITIES (second precision)
|--------Chrome/gmail.com--------|---Chrome/github.com---|--VSCode--|
08:30:00                    08:31:45                08:32:30    08:33:00

KEYSTROKES (minute aggregation)
08:30: "Hello team"     08:31: "LGTM"     08:32: "def main():"

                    ↓ RESULT (showing complete timeline) ↓

ATOMIC SLICES TABLE:
| # | Start Time | End Time | Application | URL | Keystroke Log |
|---|------------|----------|-------------|-----|---------------|
| S-001 | 08:30:00 | 08:30:59 | chrome.exe | gmail.com | Hello team |
| S-002 | 08:31:00 | 08:31:44 | chrome.exe | gmail.com | LGTM* |
| S-003 | 08:31:45 | 08:31:59 | chrome.exe | github.com | |
| S-004 | 08:32:00 | 08:32:29 | chrome.exe | github.com | def main():* |
| S-005 | 08:32:30 | 08:32:59 | vscode.exe | main.py | |
```

*Keystrokes occurred in minutes with context switches - exact attribution uncertain

## Implementation Approach

### Core Rule: Slices Bounded by Context Changes AND Minutes

An atomic slice ends when ANY of these occur:
- Application changes
- Window/URL changes within an app
- Minute boundary is reached (when keystrokes are present)

### Consolidation Rules: Optimizing for Token Efficiency

To reduce token usage while preserving data accuracy, consecutive slices within the same context are consolidated when ALL conditions are met:

1. **Within Same Context**: Slices have the same app/URL
2. **No Keystrokes**: No keyboard activity in any of the slices being consolidated
3. **No Multitasking**: Neither slice is flagged as multitasking/concurrent usage
4. **Consecutive Slices**: Slices are temporally adjacent within the entry (gap ≤ 1 second tolerance)

**How Consolidation Works**:
- Consolidation happens AFTER atomic slices are created
- Only slices within the same app/URL context can be consolidated
- Workblock entries are already bounded by context switches
- Most visible during reading/research sessions with minimal interaction

**When NOT to Consolidate**:
- Any keystroke activity (preserves attribution accuracy)
- Slices marked as multitasking/concurrent usage
- Slices in different contexts (different app/URL)
- Non-consecutive slices within an entry
- Gaps > 1 second between slices

**Note**: Consolidation is most effective for low-activity periods. Highly active users with frequent keystrokes will show minimal consolidation, which is by design to preserve accurate keystroke attribution.

### Table Display: Complete Workblock Activity with Dual Granularity

To ensure "no data lost":
- Every row represents one atomic slice
- Rows appear at context switches (second precision) AND minute boundaries
- Multiple slices can exist within one minute (rapid switching)
- Long sessions create many 1-minute slices

### Example: Multiple Slices per Minute
```
| # | Start Time | End Time | Application | URL | Keystroke Log |
|---|------------|----------|-------------|-----|---------------|
| S-001 | 09:00:00 | 09:00:14 | chrome.exe | gmail.com | Quick task* |
| S-002 | 09:00:15 | 09:00:29 | slack.exe | general | |
| S-003 | 09:00:30 | 09:00:44 | chrome.exe | docs.com | |
| S-004 | 09:00:45 | 09:00:59 | vscode.exe | main.py | |
| S-005 | 09:01:00 | 09:01:59 | vscode.exe | main.py | def test(): |
```

This shows 4 atomic slices in the first minute (15 seconds each), with ambiguous keystroke attribution.

## Output Format

### Keystroke Compression

To reduce token count, consecutive repeated keystrokes are compressed:
- `<Cmd><Cmd><Cmd>` becomes `<Cmd>{3}`
- `<Backspace><Backspace><Backspace>` becomes `<Backspace>{3}`
- `<Ctrl><Ctrl + a><Ctrl + c><Ctrl><Ctrl + a><Ctrl + c>` becomes `(<Ctrl><Ctrl + a><Ctrl + c>){2}`

### Output Table Format

The table shows the complete activity sequence with atomic slices:

```
| # | Start Time | End Time | Application | URL | Keystroke Log |
|---|------------|----------|-------------|-----|---------------|
| S-001 | 08:30:00 | 08:30:59 | chrome.exe | gmail.com | Hello team, I wanted to update you on |
| S-002 | 08:31:00 | 08:31:44 | chrome.exe | gmail.com | the project status.<Enter>{2}Best, John* |
| S-003 | 08:31:45 | 08:31:59 | chrome.exe | github.com | |
| S-004 | 08:32:00 | 08:32:29 | chrome.exe | github.com | LGTM! :shipit:* |
| S-005 | 08:32:30 | 08:32:59 | vscode.exe | main.py | |
| S-006 | 08:33:00 | 08:33:59 | vscode.exe | main.py | return sum(item.price for item in items) |
```

### Key Format Decisions (Display Layer)

Note: These are formatting conventions for display, not part of the core data structure tested in the code:

1. **Slice Number**: Sequential numbering (S-001, S-002, etc.) for display purposes
2. **Start Time/End Time**: Shows precise time boundaries for each atomic slice
3. **Application**: Application name (actual format depends on the data)
4. **URL**: URL for browsers, empty for other apps
5. **Keystroke Log**: Actual keystroke text (empty if no keystrokes)
6. **Attribution**: Asterisk (*) may mark ambiguous keystrokes in display
7. **Complete Activity Log**: Show every atomic slice to ensure "no data lost"

## Examples

### Example 1: Simple Case - One Context Per Minute

**Raw Data:**
```
Activities:
09:00:00-09:05:00  VSCode  main.py
09:05:00-09:10:00  Chrome  stackoverflow.com

Keystrokes:
09:00  "def calculate_total():" (21 keystrokes)
09:01  "    return sum(items)" (20 keystrokes)
09:02  (no keystrokes)
09:03  "# TODO: add validation" (22 keystrokes)
09:04  (no keystrokes)
09:05  (no keystrokes)
09:06  (no keystrokes)
09:07  "python list comprehension" (25 keystrokes)
```

**Generated Atomic Slices Table:**
| # | Start Time | End Time | Application | URL | Keystroke Log |
|---|------------|----------|-------------|-----|---------------|
| S-001 | 09:00:00 | 09:00:59 | vscode.exe | main.py | def calculate_total(): |
| S-002 | 09:01:00 | 09:01:59 | vscode.exe | main.py |     return sum(items) |
| S-003 | 09:02:00 | 09:02:59 | vscode.exe | main.py | |
| S-004 | 09:03:00 | 09:03:59 | vscode.exe | main.py | # TODO: add validation |
| S-005 | 09:04:00 | 09:04:59 | vscode.exe | main.py | |
| S-006 | 09:05:00 | 09:05:59 | chrome.exe | stackoverflow.com | |
| S-007 | 09:06:00 | 09:06:59 | chrome.exe | stackoverflow.com | |
| S-008 | 09:07:00 | 09:07:59 | chrome.exe | stackoverflow.com | python list comprehension |
| S-009 | 09:08:00 | 09:08:59 | chrome.exe | stackoverflow.com | |
| S-010 | 09:09:00 | 09:09:59 | chrome.exe | stackoverflow.com | |

**Interpretation**: 10 atomic slices showing minute-by-minute activity. Clean attribution since no context switches within minutes.

### Example 2: Rapid Context Switching Within Minutes

**Raw Data:**
```
Activities:
10:30:00-10:30:15  Chrome   gmail.com
10:30:15-10:30:45  Chrome   calendar.google.com  
10:30:45-10:31:20  Chrome   gmail.com
10:31:20-10:32:00  Slack    general

Keystrokes:
10:30  "Meet at 3pm?" (12 keystrokes)
10:31  "Sounds good!" (12 keystrokes)
```

**Generated Atomic Slices Table:**
| # | Start Time | End Time | Application | URL | Keystroke Log |
|---|------------|----------|-------------|-----|---------------|
| S-001 | 10:30:00 | 10:30:14 | chrome.exe | gmail.com | Meet at 3pm?* |
| S-002 | 10:30:15 | 10:30:44 | chrome.exe | calendar.google.com | |
| S-003 | 10:30:45 | 10:30:59 | chrome.exe | gmail.com | |
| S-004 | 10:31:00 | 10:31:19 | chrome.exe | gmail.com | Sounds good!* |
| S-005 | 10:31:20 | 10:31:59 | slack.exe | general | |

*Keystrokes occurred in minutes with multiple context switches

**Note**: The 10:30 minute shows 3 context switches. The "Meet at 3pm?" keystrokes could have been typed in any of those contexts. Similarly, the 10:31 keystrokes are ambiguous between Chrome/gmail and Slack.

**Interpretation**: 5 atomic slices with rapid context switching. The asterisk marking for ambiguous keystrokes is a display formatting feature, not part of the core data structure.

### Example 3: Long Reading Session with Brief Interactions

**Raw Data:**
```
Activities:
14:00:00-14:35:00  Chrome  medium.com/article

Keystrokes:
14:00  (no keystrokes)
14:01  "<Ctrl+D>" (1 keystroke - bookmarked)
14:02-14:19  (no keystrokes - reading)
14:20  "<Ctrl+C>" (1 keystroke - copied quote)
14:21-14:34  (no keystrokes - reading)
```

**Generated Atomic Slice Table (with consolidation):**
| # | Start Time | End Time | Application | URL | Keystroke Log |
|---|------------|----------|-------------|-----|---------------|
| S-001 | 14:00:00 | 14:00:59 | chrome.exe | medium.com/article | |
| S-002 | 14:01:00 | 14:01:59 | chrome.exe | medium.com/article | <Ctrl+D> |
| S-003 | 14:02:00 | 14:19:59 | chrome.exe | medium.com/article | |
| S-004 | 14:20:00 | 14:20:59 | chrome.exe | medium.com/article | <Ctrl+C> |
| S-005 | 14:21:00 | 14:34:59 | chrome.exe | medium.com/article | |

**Interpretation**: 5 atomic slices instead of 35:
- S-001: Reading (14:00:00-14:00:59) - single minute
- S-002: Bookmark action (14:01:00-14:01:59) - preserved due to keystroke
- S-003: Reading (14:02:00-14:19:59) - 18 minutes consolidated
- S-004: Copy action (14:20:00-14:20:59) - preserved due to keystroke
- S-005: Reading (14:21:00-14:34:59) - 14 minutes consolidated

**Token Savings**: 35 slices reduced to 5 (86% reduction) while preserving all keystroke attribution.

### Example 4: Cross-Application Copy-Paste Workflow

**Raw Data:**
```
Activities:
11:15:00-11:15:30  Excel     budget.xlsx
11:15:30-11:15:35  Chrome    gmail.com
11:15:35-11:16:45  Excel     budget.xlsx
11:16:45-11:17:00  Chrome    gmail.com

Keystrokes:
11:15  "<Ctrl+C>" (1 keystroke)
11:16  "=SUM(A1:A10)<Enter><Ctrl+C>" (20 keystrokes)
11:17  "The total is <Ctrl+V>" (15 keystrokes)
```

**Generated Atomic Slices Table:**
| # | Start Time | End Time | Application | URL | Keystroke Log |
|---|------------|----------|-------------|-----|---------------|
| S-001 | 11:15:00 | 11:15:29 | excel.exe | budget.xlsx | <Ctrl+C>* |
| S-002 | 11:15:30 | 11:15:34 | chrome.exe | gmail.com | |
| S-003 | 11:15:35 | 11:15:59 | excel.exe | budget.xlsx | |
| S-004 | 11:16:00 | 11:16:44 | excel.exe | budget.xlsx | =SUM(A1:A10)<Enter><Ctrl+C>* |
| S-005 | 11:16:45 | 11:16:59 | chrome.exe | gmail.com | |
| S-006 | 11:17:00 | 11:17:59 | chrome.exe | gmail.com | The total is <Ctrl+V> |

*Keystrokes in minutes with context switches

**Interpretation**: 6 atomic slices showing rapid switching for copy-paste workflow. The 11:15 and 11:16 keystrokes are marked as ambiguous due to multiple context switches in those minutes.




## Edge Cases & Decisions

### 1. Sub-Second Context Switches
If user switches contexts multiple times within one second, show each switch as a separate row with the same timestamp.

### 2. Missing Activity Data
Handled by the 4-phase matching algorithm (see technical documentation). Orphan keystrokes are grouped into synthetic activities.

### 3. Overlapping Activities
If activity data shows overlaps (data quality issue), resolve by trimming earlier activity to end when next begins.

### 4. Cross-Day Boundaries
Split slices at midnight to maintain day-based partitioning.

### 5. "Detectable Inactivity" 
The PM mentions "detectable inactivity" as a slice boundary. With current data constraints, we cannot reliably detect true inactivity (user might be reading, thinking, or using mouse). Gaps in activity data are preserved as-is. Future telemetry improvements will allow better detection of actual inactivity vs active non-keyboard work.

### 6. Implementation Notes

- Slices are created based on minute-level changes rather than second-level when no context switches occur
- The system optimizes for token efficiency while maintaining complete data fidelity
- Consolidation only happens when gaps between slices are ≤ 1 second and neither slice is marked as multitasking
- Workblocks with total duration < 5 minutes are filtered out during processing
- Display formatting is handled by the formatter, not the builder
- Active duration is calculated as the sum of all atomic slice durations and will never exceed total workblock duration
- Boundary slice creation has been removed to prevent overlapping time ranges
