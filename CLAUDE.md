# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**claude-spend** is an npm CLI tool that visualizes Claude Code token usage. It reads local session files from `~/.claude/projects/` and `~/.claude/history.jsonl`, parses JSONL session data, and serves a single-page dashboard on localhost via Express.

Run with `npx claude-spend` — no configuration needed.

## Commands

```bash
npm install          # install dependencies
npm start            # run the dashboard server (default port 3456)
node src/index.js    # same as npm start
node src/index.js --port 8080    # custom port
node src/index.js --no-open      # skip auto-opening browser
```

No build step, no tests, no linter currently configured.

## Architecture

The app has three layers, all plain Node.js (CommonJS, no transpilation):

1. **`src/index.js`** — CLI entrypoint (shebang `#!/usr/bin/env node`). Parses `--port`, `--no-open`, `--help` args, starts the Express server, auto-opens browser via the `open` package.

2. **`src/server.js`** — Express server with two API routes and static file serving:
   - `GET /api/data` — returns parsed session data (cached in memory after first call)
   - `GET /api/refresh` — clears require cache for `parser.js` and re-parses all sessions
   - Static files served from `src/public/`

3. **`src/parser.js`** — Core data processing (~550 lines). Reads `~/.claude/` directory:
   - `parseJSONLFile()` — streams JSONL files line-by-line via `readline`
   - `extractSessionData()` — pairs user messages with assistant responses, extracts token counts (input + cache_creation + cache_read), model, and tool usage
   - `parseAllSessions()` — walks `~/.claude/projects/*/` directories, aggregates into sessions, daily usage, model breakdown, per-project breakdown, top prompts, and grand totals
   - `generateInsights()` — produces 10 types of actionable insights (vague prompts, context growth, marathon sessions, input-heavy ratio, day patterns, model mismatch, tool-heavy, project dominance, conversation efficiency, heavy context)

4. **`src/public/index.html`** — Single-file SPA dashboard (~1200 lines). All HTML, CSS, and JS in one file. Fetches `/api/data`, renders charts and tables with vanilla JS (no framework). Uses Inter font and CSS custom properties for theming.

## Key Data Flow

```
~/.claude/projects/*/*.jsonl  →  parser.parseAllSessions()  →  /api/data JSON  →  dashboard renders
~/.claude/history.jsonl       →  session display names
```

The parser pairs `type: "user"` entries with subsequent `type: "assistant"` entries that contain `message.usage` token counts. It skips `isMeta` entries, `<local-command`/`<command-name` prefixed content, and `<synthetic>` model entries.

## Dependencies

Only two runtime dependencies: `express` and `open`. No dev dependencies.
