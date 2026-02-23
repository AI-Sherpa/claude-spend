# Day-on-Day Behavior Trends

## Problem

The current behavior trends compare the last 7 days against the prior 7 days. For daily Claude Code users, this is too slow to surface feedback — a bad habit today won't show up for a week.

## Design

Change the comparison windows in `computeBehaviorTrends()`:

- **Recent** = today's sessions (midnight local time to now)
- **Prior** = rolling 7-day average (past 7 days, excluding today)

The 8 tracked metrics, `computeMetric()` helper, and all dashboard rendering stay unchanged. The data shape returned by `computeBehaviorTrends()` is identical — only the session filtering and window metadata change.

## Changes

### parser.js — `computeBehaviorTrends()`

1. Replace cutoff calculations (lines 575-579): use start-of-today and 7-days-ago instead of 7/14-day windows.
2. Update the "not enough data" guard (line 581): require today sessions + at least 1 prior day session.
3. Update `window` metadata (lines 677-679): reflect "today" and "7-day avg" labels.

### index.html — dashboard text

1. Update "not enough data" message (line 964): change "14 days" to reflect new requirement.
2. Update banner subtitle (around line 1024): say "today vs 7-day avg" instead of "week-on-week".

## What stays the same

- All 8 metrics and their computation functions
- `computeMetric()` threshold and direction logic
- Trend badges on insight cards
- All CSS styling
- Dashboard metric card rendering
- API response shape
