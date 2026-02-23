# Behavior Change Tracking Design

## Problem
The current insight system produces static snapshots — it flags patterns like "vague prompts cost you tokens" but doesn't track whether the user's behavior is improving or worsening over time. Users have no feedback loop to know if acting on recommendations actually helped.

## Solution: Metric-per-Insight Trends (Approach A)
Each existing insight gets a paired "behavior metric" tracked over a rolling 7-day window. The parser computes the metric for the recent 7 days vs the prior 7 days and attaches trend data.

## Time Window
- **Method:** Rolling recent 7 days vs prior 7 days
- **Minimum data:** 14 days of usage required for trends to appear
- **Edge cases:** No recent sessions = "no data" (not "improved to zero")

## Tracked Metrics

| Metric ID | Paired Insight | What it measures | "Improving" means |
|-----------|---------------|-----------------|-------------------|
| `prompt-specificity` | `vague-prompts` | Avg character length of user prompts | Getting longer |
| `session-length` | `context-growth`, `marathon-sessions` | Avg turns per session | Getting shorter |
| `clear-frequency` | `context-growth` | `/clear` commands per session | More frequent clearing |
| `tokens-per-turn` | `conversation-efficiency` | Avg tokens per message | Lower cost per turn |
| `output-ratio` | `input-heavy` | Output tokens as % of total | Higher output ratio |
| `tool-ratio` | `tool-heavy` | Tool calls per user message | Lower ratio |
| `startup-context` | `heavy-context` | Avg tokens in first message | Lower startup cost |
| `opus-simple-ratio` | `model-mismatch` | % of simple tasks using Opus | Fewer Opus for simple tasks |

## Data Layer Changes (parser.js)

### 1. Count /clear commands
Currently `extractSessionData()` skips `<command-name` prefixed entries. Add a parallel counter that captures `/clear` occurrences per session with timestamps. No change to existing token logic.

### 2. New `computeBehaviorTrends()` function
- Takes full sessions array
- Splits into "recent 7 days" and "prior 7 days" windows
- Computes each metric for both windows
- Returns trend objects: `{ current, previous, change, direction, pctChange }`

### 3. Attach trends to insights
Each insight object gets an optional `trend` field mapping to its paired metric.

### 4. New `behaviorTrends` in API response
Top-level field with all metric trends + aggregate: `{ improved: N, worsened: N, stable: N, overallDirection }`

## UI Layer Changes (index.html)

### 1. Trend badges on insight cards
Small pill next to each insight title showing trend direction and magnitude.

### 2. New "Behavior Trends" section
Between Insights and Charts sections:
- Summary banner: "X of Y tracked behaviors improved this week"
- Metric cards: compact row showing metric name, current value, trend arrow, % change

## Edge Cases
- <14 days of data: Show "Not enough data yet" message
- No sessions in recent window: Treated as "no data"
- Insight not triggered but metric tracked: Trend card still appears
- Metric has zero denominator (e.g., no sessions): Marked as "no data"
