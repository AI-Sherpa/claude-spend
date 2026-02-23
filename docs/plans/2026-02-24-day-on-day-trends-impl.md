# Day-on-Day Behavior Trends Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Change behavior trends from week-on-week (7d vs 7d) to day-on-day (today vs 7-day avg) comparison.

**Architecture:** Modify the window calculations in `computeBehaviorTrends()` to filter today's sessions as "recent" and the past 7 days (excluding today) as "prior". Update dashboard text to match. No new files, no new dependencies, no shape changes.

**Tech Stack:** Node.js (CommonJS), vanilla JS frontend

---

### Task 1: Update parser window calculations

**Files:**
- Modify: `src/parser.js:573-579`

**Step 1: Replace the cutoff variables and session filters**

Change lines 573-579 from:

```js
  const now = new Date();
  const msPerDay = 86400000;
  const recentCutoff = new Date(now - 7 * msPerDay);
  const priorCutoff = new Date(now - 14 * msPerDay);

  const recent = sessions.filter(s => s.timestamp && new Date(s.timestamp) >= recentCutoff);
  const prior = sessions.filter(s => s.timestamp && new Date(s.timestamp) >= priorCutoff && new Date(s.timestamp) < recentCutoff);
```

To:

```js
  const now = new Date();
  const startOfToday = new Date(now.getFullYear(), now.getMonth(), now.getDate());
  const msPerDay = 86400000;
  const priorCutoff = new Date(startOfToday - 7 * msPerDay);

  const recent = sessions.filter(s => s.timestamp && new Date(s.timestamp) >= startOfToday);
  const prior = sessions.filter(s => s.timestamp && new Date(s.timestamp) >= priorCutoff && new Date(s.timestamp) < startOfToday);
```

**Step 2: Verify the server starts cleanly**

Run: `node src/index.js --no-open &` then `curl -s http://localhost:3456/api/data | node -e "process.stdin.resume(); let d=''; process.stdin.on('data',c=>d+=c); process.stdin.on('end',()=>{const j=JSON.parse(d); console.log('hasData:', j.behaviorTrends.hasData); console.log('window:', JSON.stringify(j.behaviorTrends.window,null,2))})"`

Expected: `hasData: true` (if you have today + prior sessions) or `hasData: false`. The `window.recent.from` should show today's date.

Kill the server after.

**Step 3: Commit**

```bash
git add src/parser.js
git commit -m "feat: change behavior trends to today vs 7-day avg"
```

---

### Task 2: Update parser window metadata

**Files:**
- Modify: `src/parser.js:676-679`

**Step 1: Update the window labels in the return value**

Change lines 676-679 from:

```js
    window: {
      recent: { from: recentCutoff.toISOString().split('T')[0], to: now.toISOString().split('T')[0], sessions: recent.length },
      prior: { from: priorCutoff.toISOString().split('T')[0], to: recentCutoff.toISOString().split('T')[0], sessions: prior.length },
    },
```

To:

```js
    window: {
      recent: { from: startOfToday.toISOString().split('T')[0], to: now.toISOString().split('T')[0], sessions: recent.length, label: 'today' },
      prior: { from: priorCutoff.toISOString().split('T')[0], to: startOfToday.toISOString().split('T')[0], sessions: prior.length, label: '7-day avg' },
    },
```

**Step 2: Commit**

```bash
git add src/parser.js
git commit -m "feat: add window labels to behavior trends metadata"
```

---

### Task 3: Update dashboard "no data" message

**Files:**
- Modify: `src/public/index.html:964`

**Step 1: Update the no-data text**

Change line 964 from:

```js
    noData.textContent = 'Not enough data yet — need at least 14 days of usage to show behavior trends.';
```

To:

```js
    noData.textContent = 'Not enough data yet — need sessions today and at least one prior day to show behavior trends.';
```

**Step 2: Commit**

```bash
git add src/public/index.html
git commit -m "fix: update behavior trends no-data message for day-on-day"
```

---

### Task 4: Update dashboard banner text

**Files:**
- Modify: `src/public/index.html:973-978, 1024, 1056`

**Step 1: Update banner text from "this week" to "today"**

Change lines 973-977 from:

```js
  const bannerText = s.overallDirection === 'improving'
    ? s.improved + ' of ' + s.total + ' tracked behaviors improved this week'
    : s.overallDirection === 'worsening'
    ? s.worsened + ' of ' + s.total + ' tracked behaviors worsened this week'
    : 'Your usage patterns are mostly stable this week';
```

To:

```js
  const bannerText = s.overallDirection === 'improving'
    ? s.improved + ' of ' + s.total + ' tracked behaviors improved today'
    : s.overallDirection === 'worsening'
    ? s.worsened + ' of ' + s.total + ' tracked behaviors worsened today'
    : 'Your usage patterns are mostly stable today';
```

**Step 2: Update banner subtitle from "this week/last week" to "today/7-day avg"**

Change line 1024 from:

```js
  bannerSubEl.textContent = bannerSub + ' · ' + bt.window.recent.sessions + ' sessions this week, ' + bt.window.prior.sessions + ' last week';
```

To:

```js
  bannerSubEl.textContent = bannerSub + ' · ' + bt.window.recent.sessions + ' sessions today vs ' + bt.window.prior.sessions + ' sessions (7-day avg)';
```

**Step 3: Update metric card "was X last week" to "7-day avg: X"**

Change line 1056 from:

```js
    prev.textContent = 'was ' + formatPrevValue(m) + ' last week';
```

To:

```js
    prev.textContent = '7-day avg: ' + formatPrevValue(m);
```

**Step 4: Commit**

```bash
git add src/public/index.html
git commit -m "feat: update dashboard text for day-on-day trends"
```

---

### Task 5: Smoke test the full flow

**Step 1: Start the server and verify the dashboard loads**

Run: `node src/index.js --no-open &`
Then: `curl -s http://localhost:3456/api/data | node -e "process.stdin.resume(); let d=''; process.stdin.on('data',c=>d+=c); process.stdin.on('end',()=>{const j=JSON.parse(d); const bt=j.behaviorTrends; console.log('hasData:', bt.hasData); if(bt.window){console.log('recent label:', bt.window.recent.label); console.log('prior label:', bt.window.prior.label); console.log('recent sessions:', bt.window.recent.sessions); console.log('metrics:', bt.metrics.length)}})"`

Expected: `recent label: today`, `prior label: 7-day avg`.

Kill the server after.
