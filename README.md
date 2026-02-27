# claude-spend

See where your Claude Code tokens go. One command, zero setup.

## Problem

 I've been using Claude Code every day for 3 months. I hit the usage limit almost daily, but had zero visibility into which prompts were eating my tokens. So I built claude-spend. One command, zero setup. 

## How does it look

### Light Theme
Clean, minimal interface with full visibility into your token spending.

![Claude Spend Dashboard - Light Theme](dashboard-light-overview.png)

### Dark Theme
Dark theme with adaptive colors for comfortable viewing in low-light environments.

![Claude Spend Dashboard - Dark Theme](dashboard-dark-overview.png)



## Install

```
npx claude-spend
```

That's it. Opens a dashboard in your browser.


## What it does

- **Reads your local Claude Code session files** (nothing leaves your machine)
- **Tracks token usage** across conversations, daily trends, and per-model breakdown
  - Input, output, and cache tokens
  - Cache creation and read metrics
  - Token efficiency per session
- **Intelligent project organization** with automatic time-based grouping (Today, Yesterday, Inactive)
- **Query-level tracking** showing conversation activity alongside token metrics
- **Surfaces actionable insights** like which prompts cost the most, usage patterns, and optimization opportunities
- **Interactive dashboard** with charts, tables, theme toggle (light/dark), and real-time refresh


## How it works

`claude-spend` reads your local Claude Code session data from:
- `~/.claude/projects/*/` — All project session files
- `~/.claude/history.jsonl` — Session metadata

It then:
1. **Parses JSONL files** line-by-line to extract user messages and assistant responses
2. **Extracts token metrics** — input tokens, output tokens, cache creation/read counts, and model info
3. **Aggregates data** into daily stats, per-project breakdowns, and usage trends
4. **Generates insights** — 10 types of analysis including efficiency metrics, context growth patterns, and time-of-day trends
5. **Serves a dashboard** on localhost for interactive exploration

All processing happens locally on your machine. No data leaves your computer.

### Data Flow

```
~/.claude/projects/*/*.jsonl  →  Parse & aggregate  →  Express API  →  Web dashboard
~/.claude/history.jsonl       →  (token metrics)    →  /api/data   →  (interactive charts)
```


## Dashboard Sections

### Overview
- **Grand totals** — Lifetime statistics showing:
  - Total tokens spent across all sessions
  - Total sessions and queries executed
  - Average tokens per session
- **Daily usage chart** — Visual trend of token consumption over time with interactive hover details
- **Model breakdown** — Token consumption by Claude model version with percentage breakdown
- **Theme toggle** — Switch between light and dark themes for comfortable viewing at any time of day

### Projects
- **Smart Time-Based Grouping** — Projects automatically grouped into:
  - **Today** — Active projects from the current day with session and query counts
  - **Yesterday** — Recently used projects from the previous day
  - **Inactive** — All other projects organized by historical activity
- **Per-project stats** — For each project, view:
  - Total tokens spent (input + output)
  - Number of sessions and queries
  - Last active timestamp
- **Top prompts** — Most frequently used prompts from recent sessions with token costs

### Insights
Auto-generated analysis including:
- **Session efficiency** — Cost per query, token efficiency
- **Context growth** — How conversation context size evolves over time
- **Usage patterns** — Peak usage times, day-of-week trends
- **Model utilization** — Which models you're using and optimization opportunities
- **Tool usage** — API calls and tool integration patterns

### Trends
- **Daily stats** — Session counts and tokens by day
- **Weekly patterns** — Usage distribution across weekdays
- **Model timeline** — Historical adoption of different Claude models


## Options

```
claude-spend --port 8080   # custom port (default: 3456)
claude-spend --no-open     # don't auto-open browser
claude-spend --help        # show all options
```

## Technical Details

**Stack:**
- **Node.js** — CommonJS (no build step, no transpilation)
- **Express** — Lightweight server with two API routes
- **Vanilla JavaScript** — Single-page app (no framework dependencies)
- **CSS variables** — Dark/light theme support

**Architecture:**
- `src/index.js` — CLI entry point with argument parsing
- `src/server.js` — Express server with `/api/data` and `/api/refresh` routes
- `src/parser.js` — Core parsing logic (~550 lines)
  - `parseJSONLFile()` — Streams JSONL files line-by-line
  - `extractSessionData()` — Pairs user/assistant messages and extracts token counts
  - `parseAllSessions()` — Walks project directories and aggregates stats
  - `generateInsights()` — Produces 10 types of actionable insights
- `src/public/index.html` — Interactive dashboard (~1200 lines)

**Dependencies:**
- `express` — Web server
- `open` — Auto-open browser

No dev dependencies, no linter, no tests. The project prioritizes simplicity and quick iteration.


## Development

Since there's no build step, changes are instant:

1. Modify files in `src/`
2. Refresh your browser
3. Use `/api/refresh` endpoint to force the parser to re-read all session files

```bash
npm install    # Install dependencies
npm start      # Run the dashboard (port 3456)
node src/index.js --port 8080   # Custom port
```

**Common tasks:**
- **Add new chart** — Edit `src/public/index.html` and fetch from `/api/data`
- **Change insights** — Modify `generateInsights()` in `src/parser.js`
- **Add API endpoint** — Add route in `src/server.js`


## Privacy & Security

All data stays local. `claude-spend` reads files from `~/.claude/` on your machine and serves a dashboard on `localhost` only. **No data is sent anywhere.**

- 🔒 Zero external network calls
- 🔒 No analytics, tracking, or telemetry
- 🔒 All processing happens on your computer
- 🔒 Minimal dependencies (only `express` and `open`)

For security concerns, see [SECURITY.md](./SECURITY.md).

## License

MIT
