# Behavior Change Tracking Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add behavior trend tracking that compares rolling 7-day windows to show whether user behavior around token optimization is improving or worsening.

**Architecture:** Extend the parser with `/clear` command counting + a new `computeBehaviorTrends()` function that computes 8 metrics across two time windows. Attach trend data to existing insights and add a new `behaviorTrends` field to the API response. Dashboard gets trend badges on insight cards and a new Behavior Trends section.

**Tech Stack:** Plain Node.js (CommonJS), vanilla JS/HTML/CSS — no new dependencies.

**Security note:** All data rendered in the dashboard uses the existing `escapeHtml()` function for user-generated content. The dashboard is a local-only tool served on localhost, and all content originates from the user's own session files. The existing pattern of building HTML strings with `escapeHtml()` for any user-originated text (prompts, project names) is maintained throughout.

---

### Task 1: Count /clear commands in extractSessionData

**Files:**
- Modify: `src/parser.js:26-79` (extractSessionData function)

**Step 1: Add clearCount tracking to extractSessionData**

In `extractSessionData()`, before the main loop, add a `clearCount` variable. Inside the loop, before the existing skip logic for `<command-name`, check if the content contains `/clear` and increment the counter. Return `clearCount` alongside `queries`.

Change the function signature to return an object instead of just the queries array:

```js
function extractSessionData(entries) {
  const queries = [];
  let pendingUserMessage = null;
  let clearCount = 0;

  for (const entry of entries) {
    if (entry.type === 'user' && entry.message?.role === 'user') {
      const content = entry.message.content;
      if (entry.isMeta) continue;

      // Count /clear commands before skipping them
      if (typeof content === 'string' && content.startsWith('<command-name')) {
        if (content.includes('/clear')) clearCount++;
        continue;
      }
      if (typeof content === 'string' && content.startsWith('<local-command')) continue;

      const textContent = typeof content === 'string'
        ? content
        : content.filter(b => b.type === 'text').map(b => b.text).join('\n').trim();
      pendingUserMessage = {
        text: textContent || null,
        timestamp: entry.timestamp,
      };
    }

    // ... rest of assistant handling unchanged ...
  }

  return { queries, clearCount };
}
```

**Step 2: Update all callers of extractSessionData**

In `parseAllSessions()` at line 128, change:
```js
const queries = extractSessionData(entries);
if (queries.length === 0) continue;
```
to:
```js
const { queries, clearCount } = extractSessionData(entries);
if (queries.length === 0) continue;
```

**Step 3: Add clearCount to session objects**

At line 181 in the `sessions.push(...)` call, add `clearCount` to the session object:
```js
sessions.push({
  sessionId,
  project: projectDir,
  date,
  timestamp: firstTimestamp,
  firstPrompt: firstPrompt.substring(0, 200),
  model: primaryModel,
  queryCount: queries.length,
  queries,
  clearCount,
  inputTokens,
  outputTokens,
  totalTokens,
});
```

**Step 4: Verify the server still starts and returns data**

Run: `node src/index.js --no-open &` then `curl -s http://localhost:3456/api/data | node -e "process.stdin.on('data',d=>{const j=JSON.parse(d);console.log('sessions:',j.sessions.length);const withClears=j.sessions.filter(s=>s.clearCount>0);console.log('sessions with /clear:',withClears.length)})"`

Expected: Server starts, returns session data, `clearCount` field present on sessions.

**Step 5: Commit**

```bash
git add src/parser.js
git commit -m "feat: count /clear commands per session in parser"
```

---

### Task 2: Implement computeBehaviorTrends function

**Files:**
- Modify: `src/parser.js` (add new function after generateInsights, before fmt)

**Step 1: Add the computeBehaviorTrends function**

Insert this function before the `fmt()` function (before line 543):

```js
function computeBehaviorTrends(sessions) {
  const now = new Date();
  const msPerDay = 86400000;
  const recentCutoff = new Date(now - 7 * msPerDay);
  const priorCutoff = new Date(now - 14 * msPerDay);

  const recent = sessions.filter(s => s.timestamp && new Date(s.timestamp) >= recentCutoff);
  const prior = sessions.filter(s => s.timestamp && new Date(s.timestamp) >= priorCutoff && new Date(s.timestamp) < recentCutoff);

  // Not enough data for two windows
  if (recent.length === 0 || prior.length === 0) {
    return { metrics: [], summary: null, hasData: false };
  }

  function computeMetric(id, label, recentVal, priorVal, lowerIsBetter) {
    if (priorVal === null || recentVal === null) {
      return { id, label, current: recentVal, previous: priorVal, direction: 'no-data', pctChange: 0 };
    }
    const diff = recentVal - priorVal;
    const pctChange = priorVal !== 0 ? Math.round((diff / Math.abs(priorVal)) * 100) : 0;
    const threshold = 5; // % change must exceed this to count as improved/worsened
    let direction = 'stable';
    if (Math.abs(pctChange) > threshold) {
      const rawBetter = lowerIsBetter ? diff < 0 : diff > 0;
      direction = rawBetter ? 'improved' : 'worsened';
    }
    return { id, label, current: Math.round(recentVal * 10) / 10, previous: Math.round(priorVal * 10) / 10, direction, pctChange };
  }

  function avgPromptLength(sessionsArr) {
    let total = 0, count = 0;
    for (const s of sessionsArr) {
      for (const q of s.queries) {
        if (q.userPrompt) { total += q.userPrompt.length; count++; }
      }
    }
    return count > 0 ? total / count : null;
  }

  function avgSessionLength(sessionsArr) {
    if (sessionsArr.length === 0) return null;
    return sessionsArr.reduce((s, ses) => s + ses.queryCount, 0) / sessionsArr.length;
  }

  function clearFrequency(sessionsArr) {
    if (sessionsArr.length === 0) return null;
    return sessionsArr.reduce((s, ses) => s + (ses.clearCount || 0), 0) / sessionsArr.length;
  }

  function tokensPerTurn(sessionsArr) {
    let totalTokens = 0, totalQueries = 0;
    for (const s of sessionsArr) { totalTokens += s.totalTokens; totalQueries += s.queryCount; }
    return totalQueries > 0 ? totalTokens / totalQueries : null;
  }

  function outputRatio(sessionsArr) {
    let totalOut = 0, totalAll = 0;
    for (const s of sessionsArr) { totalOut += s.outputTokens; totalAll += s.totalTokens; }
    return totalAll > 0 ? (totalOut / totalAll) * 100 : null;
  }

  function toolRatio(sessionsArr) {
    let totalTools = 0, totalUserMsgs = 0;
    for (const s of sessionsArr) {
      const userMsgs = s.queries.filter(q => q.userPrompt).length;
      totalUserMsgs += userMsgs;
      totalTools += s.queryCount - userMsgs;
    }
    return totalUserMsgs > 0 ? totalTools / totalUserMsgs : null;
  }

  function avgStartupContext(sessionsArr) {
    const starts = sessionsArr.filter(s => s.queries.length > 0).map(s => s.queries[0].inputTokens);
    return starts.length > 0 ? starts.reduce((a, b) => a + b, 0) / starts.length : null;
  }

  function opusSimpleRatio(sessionsArr) {
    const opusSess = sessionsArr.filter(s => s.model.includes('opus'));
    if (opusSess.length === 0) return null;
    const simple = opusSess.filter(s => s.queryCount < 10 && s.totalTokens < 200000);
    return (simple.length / opusSess.length) * 100;
  }

  const metrics = [
    computeMetric('prompt-specificity', 'Prompt specificity', avgPromptLength(recent), avgPromptLength(prior), false),
    computeMetric('session-length', 'Avg session length', avgSessionLength(recent), avgSessionLength(prior), true),
    computeMetric('clear-frequency', '/clear usage per session', clearFrequency(recent), clearFrequency(prior), false),
    computeMetric('tokens-per-turn', 'Tokens per message', tokensPerTurn(recent), tokensPerTurn(prior), true),
    computeMetric('output-ratio', 'Output token ratio', outputRatio(recent), outputRatio(prior), false),
    computeMetric('tool-ratio', 'Tool calls per message', toolRatio(recent), toolRatio(prior), true),
    computeMetric('startup-context', 'Startup context size', avgStartupContext(recent), avgStartupContext(prior), true),
    computeMetric('opus-simple-ratio', 'Opus for simple tasks', opusSimpleRatio(recent), opusSimpleRatio(prior), true),
  ].filter(m => m.direction !== 'no-data');

  const improved = metrics.filter(m => m.direction === 'improved').length;
  const worsened = metrics.filter(m => m.direction === 'worsened').length;
  const stable = metrics.filter(m => m.direction === 'stable').length;
  let overallDirection = 'stable';
  if (improved > worsened) overallDirection = 'improving';
  else if (worsened > improved) overallDirection = 'worsening';

  return {
    metrics,
    summary: { improved, worsened, stable, total: metrics.length, overallDirection },
    hasData: true,
    window: {
      recent: { from: recentCutoff.toISOString().split('T')[0], to: now.toISOString().split('T')[0], sessions: recent.length },
      prior: { from: priorCutoff.toISOString().split('T')[0], to: recentCutoff.toISOString().split('T')[0], sessions: prior.length },
    },
  };
}
```

**Step 2: Verify syntax**

Run: `node -e "require('./src/parser')"`

Expected: No errors.

**Step 3: Commit**

```bash
git add src/parser.js
git commit -m "feat: add computeBehaviorTrends function for rolling 7-day comparison"
```

---

### Task 3: Wire trends into parseAllSessions and attach to insights

**Files:**
- Modify: `src/parser.js:327-338` (end of parseAllSessions, return statement)

**Step 1: Call computeBehaviorTrends and build the insight-to-trend map**

After line 328 (`const insights = generateInsights(...)`) and before the return statement, add:

```js
  // Compute behavior trends
  const behaviorTrends = computeBehaviorTrends(sessions);

  // Attach trend data to matching insights
  if (behaviorTrends.hasData) {
    const insightToMetric = {
      'vague-prompts': 'prompt-specificity',
      'context-growth': 'session-length',
      'marathon-sessions': 'session-length',
      'conversation-efficiency': 'tokens-per-turn',
      'input-heavy': 'output-ratio',
      'tool-heavy': 'tool-ratio',
      'heavy-context': 'startup-context',
      'model-mismatch': 'opus-simple-ratio',
    };
    const metricMap = {};
    for (const m of behaviorTrends.metrics) metricMap[m.id] = m;
    for (const insight of insights) {
      const metricId = insightToMetric[insight.id];
      if (metricId && metricMap[metricId]) {
        insight.trend = metricMap[metricId];
      }
    }
  }
```

**Step 2: Add behaviorTrends to the return object**

Change the return statement to include `behaviorTrends`:

```js
  return {
    sessions,
    dailyUsage,
    modelBreakdown: Object.values(modelMap),
    projectBreakdown,
    topPrompts,
    totals: grandTotals,
    insights,
    behaviorTrends,
  };
```

**Step 3: Verify the API returns trend data**

Run: `node src/index.js --no-open &` then `curl -s http://localhost:3456/api/data | node -e "process.stdin.on('data',d=>{const j=JSON.parse(d);console.log('Keys:',Object.keys(j));console.log('behaviorTrends:',JSON.stringify(j.behaviorTrends,null,2).substring(0,500))})"`

Expected: `behaviorTrends` key present with `hasData`, `metrics`, `summary`, `window` fields.

**Step 4: Commit**

```bash
git add src/parser.js
git commit -m "feat: wire behavior trends into API response and attach to insights"
```

---

### Task 4: Add trend badge CSS to dashboard

**Files:**
- Modify: `src/public/index.html:253-294` (insights CSS section)

**Step 1: Add CSS for trend badges and the behavior trends section**

After the existing `.insight-action-tip` CSS block (around line 294), add the CSS for trend badges (`.trend-badge`, `.trend-badge.improved`, `.trend-badge.worsened`, `.trend-badge.stable`) and the behavior trends section (`.behavior-trends`, `.trends-banner`, `.trends-grid`, `.trend-card`, etc.). See the design doc for the full list of classes needed.

**Step 2: Commit**

```bash
git add src/public/index.html
git commit -m "feat: add CSS for trend badges and behavior trends section"
```

---

### Task 5: Add Behavior Trends section HTML to dashboard

**Files:**
- Modify: `src/public/index.html:625-627` (between insights section and charts section)

**Step 1: Add the HTML section**

After the closing `</div>` of the insights section (line 625) and before the `<!-- Charts -->` comment (line 627), insert the Behavior Trends section with an `id="behaviorTrendsSection"` div containing a section header and an `id="behaviorTrendsContent"` div for dynamic content.

**Step 2: Commit**

```bash
git add src/public/index.html
git commit -m "feat: add behavior trends section HTML to dashboard"
```

---

### Task 6: Add renderBehaviorTrends JS function

**Files:**
- Modify: `src/public/index.html` (JS section, after renderInsights function ~line 867)

**Step 1: Add the renderBehaviorTrends function**

After the `renderInsights()` function, add a `renderBehaviorTrends()` function that:
1. Reads `DATA.behaviorTrends`
2. If `!hasData`, shows a "not enough data" message
3. If `hasData`, renders a summary banner showing overall direction + counts, then a grid of trend metric cards each showing: label, current value (formatted), % change with arrow, and previous value

All user-originated content must be passed through `escapeHtml()`. Metric labels are hardcoded strings from the parser, not user input.

**Step 2: Call renderBehaviorTrends in the render function**

In the `render()` function (around line 810), add `renderBehaviorTrends()` after `renderInsights()`.

**Step 3: Commit**

```bash
git add src/public/index.html
git commit -m "feat: add renderBehaviorTrends function and wire into render()"
```

---

### Task 7: Add trend badges to insight cards

**Files:**
- Modify: `src/public/index.html` (renderInsights function, ~line 843-867)

**Step 1: Update renderInsights to show trend badges**

In the `renderInsights()` function, modify the insight card rendering to include a trend badge when the insight has a `trend` field. Add a `trendHtml` variable that generates a `<span class="trend-badge ...">` element with the direction arrow and % change. Insert it between the insight title and the expand arrow in the `.insight-top` div.

**Step 2: Commit**

```bash
git add src/public/index.html
git commit -m "feat: add trend badges to insight cards"
```

---

### Task 8: Manual verification and final commit

**Step 1: Start the server and verify visually**

Run: `node src/index.js --no-open`

Open `http://localhost:3456` in the browser. Check:
- Insights section still renders correctly
- If 14+ days of data exist: Behavior Trends section appears between Insights and Charts
- If less than 14 days of data exist: "Not enough data yet" message appears
- Trend badges appear on insight cards that have matching metrics
- Trend cards in the Behavior Trends section show metric name, current value, % change
- Summary banner shows "X of Y tracked behaviors improved/worsened"

**Step 2: Verify API response shape**

Run: `curl -s http://localhost:3456/api/data | node -e "process.stdin.on('data',d=>{const j=JSON.parse(d);console.log('Keys:',Object.keys(j));console.log('behaviorTrends:',JSON.stringify(j.behaviorTrends,null,2).substring(0,500))})"`

Expected: `behaviorTrends` key present with `hasData`, `metrics`, `summary`, `window` fields.

**Step 3: Check for regressions**

Verify existing sections (Stats, Charts, Projects, Top Prompts, All Sessions) still render correctly.

**Step 4: Final commit if any fixups needed**

```bash
git add -A
git commit -m "fix: address any issues found during manual verification"
```
