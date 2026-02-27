# Dark Theme Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add a dark theme to the Claude Spend dashboard with system preference detection and manual FAB toggle.

**Architecture:** Single-file HTML changes using CSS custom properties override with `:root[data-theme="dark"]`, system detection via `prefers-color-scheme`, and a small vanilla JS toggle script. No additional dependencies.

**Tech Stack:** HTML, CSS custom properties, vanilla JavaScript, localStorage API

---

### Task 1: Add Dark Color Palette CSS Variables

**File:** `src/public/index.html`
**Modify:** `:root` selector (lines 12-49)

Add a new `:root[data-theme="dark"]` rule set right after the light theme `:root` block. This will override all color variables when `data-theme="dark"` is set on the html element.

**Step 1: Add dark palette rule set after line 49**

Find the closing `}` of the `:root` selector (around line 49), then add this entire dark palette block immediately after:

```css
  :root[data-theme="dark"] {
    --bg: #0F172A;
    --bg-mesh-1: #1A1F35;
    --bg-mesh-2: #1A2B2A;
    --white: #1E293B;
    --text: #F1F5F9;
    --text-secondary: #CBD5E1;
    --text-tertiary: #64748B;
    --border: rgba(255,255,255,0.08);
    --border-strong: rgba(255,255,255,0.12);

    --indigo: #818CF8;
    --violet: #A78BFA;
    --purple: #C084FC;
    --teal: #2DD4BF;
    --cyan: #22D3EE;
    --emerald: #34D399;
    --amber: #FBBF24;
    --orange: #FB923C;
    --rose: #FB7185;
    --blue: #60A5FA;

    --gradient-main: linear-gradient(135deg, #818CF8, #A78BFA, #C084FC);
    --gradient-teal: linear-gradient(135deg, #2DD4BF, #22D3EE);
    --gradient-warm: linear-gradient(135deg, #FBBF24, #FB923C);
    --gradient-rose: linear-gradient(135deg, #FB7185, #F472B6);

    --shadow-sm: 0 1px 2px rgba(0,0,0,0.3);
    --shadow-md: 0 4px 16px rgba(0,0,0,0.4);
    --shadow-lg: 0 8px 32px rgba(0,0,0,0.5);
    --shadow-glow: 0 0 40px rgba(129,140,248,0.2);
  }
```

**Step 2: Update mesh gradient for dark mode**

In the `body::before` pseudo-element (around lines 61-69), update the radial gradients for dark mode. Replace the current block with:

```css
  /* Mesh gradient background */
  body::before {
    content: '';
    position: fixed; top: 0; left: 0; right: 0; height: 600px;
    background:
      radial-gradient(ellipse 80% 60% at 10% 0%, rgba(129,140,248,0.08) 0%, transparent 60%),
      radial-gradient(ellipse 60% 50% at 90% 10%, rgba(167,139,250,0.06) 0%, transparent 50%),
      radial-gradient(ellipse 50% 40% at 50% 20%, rgba(45,212,191,0.04) 0%, transparent 50%);
    pointer-events: none; z-index: 0;
  }
```

This reduces mesh opacity for dark mode to avoid overwhelming the interface.

**Step 3: Verify CSS variables are correct**

Open the HTML file in an editor and visually confirm the dark palette block is properly formatted and closed with `}`.

**Step 4: Commit**

```bash
git add src/public/index.html
git commit -m "feat: add dark theme CSS variables and mesh gradients"
```

---

### Task 2: Add Smooth Color Transitions

**File:** `src/public/index.html`
**Modify:** `body` selector (around lines 51-58)

Add transition property to body so color changes are smooth when toggling themes.

**Step 1: Update body selector**

Find the `body { ... }` rule (lines 51-58) and add `transition: background-color 300ms, color 300ms;` after the `overflow-x: hidden;` line:

```css
  body {
    font-family: var(--font);
    background: var(--bg);
    color: var(--text);
    line-height: 1.6;
    min-height: 100vh;
    overflow-x: hidden;
    transition: background-color 300ms, color 300ms;
  }
```

**Step 2: Commit**

```bash
git add src/public/index.html
git commit -m "feat: add smooth color transitions on theme toggle"
```

---

### Task 3: Add FAB Button HTML

**File:** `src/public/index.html`
**Modify:** End of body, before closing `</body>` tag

Add the FAB button HTML. Find the closing `</body>` tag and add this right before it:

**Step 1: Insert FAB button HTML**

Locate the closing `</body>` tag (at the very end of the file) and add:

```html
  <!-- Theme Toggle FAB -->
  <button id="theme-toggle-fab" class="theme-fab" title="Toggle theme" aria-label="Toggle theme">
    <svg class="theme-icon sun-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
      <circle cx="12" cy="12" r="5"></circle>
      <line x1="12" y1="1" x2="12" y2="3"></line>
      <line x1="12" y1="21" x2="12" y2="23"></line>
      <line x1="4.22" y1="4.22" x2="5.64" y2="5.64"></line>
      <line x1="18.36" y1="18.36" x2="19.78" y2="19.78"></line>
      <line x1="1" y1="12" x2="3" y2="12"></line>
      <line x1="21" y1="12" x2="23" y2="12"></line>
      <line x1="4.22" y1="19.78" x2="5.64" y2="18.36"></line>
      <line x1="18.36" y1="5.64" x2="19.78" y2="4.22"></line>
    </svg>
    <svg class="theme-icon moon-icon hidden" viewBox="0 0 24 24" fill="currentColor">
      <path d="M21 12.79A9 9 0 1 1 11.21 3 7 7 0 0 0 21 12.79z"></path>
    </svg>
    <svg class="theme-icon system-icon hidden" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
      <rect x="2" y="3" width="20" height="14" rx="2" ry="2"></rect>
      <line x1="8" y1="21" x2="16" y2="21"></line>
      <line x1="12" y1="17" x2="12" y2="21"></line>
    </svg>
  </button>
```

**Step 2: Commit**

```bash
git add src/public/index.html
git commit -m "feat: add theme toggle FAB button HTML"
```

---

### Task 4: Add FAB Button Styling

**File:** `src/public/index.html`
**Modify:** Style section, after the loading styles (around line 83)

Add complete FAB styling. Find the line with `.loading-text { ... }` (line 82) and add the FAB styles right after it.

**Step 1: Add FAB CSS**

After the `.loading-text` rule, add:

```css

  /* Theme Toggle FAB */
  .theme-fab {
    position: fixed;
    bottom: 24px;
    right: 24px;
    width: 56px;
    height: 56px;
    border-radius: 50%;
    border: none;
    background: var(--white);
    color: var(--text);
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    box-shadow: var(--shadow-lg);
    z-index: 1000;
    transition: all 300ms ease-out;
    padding: 0;
  }

  .theme-fab:hover {
    transform: scale(1.1);
    box-shadow: var(--shadow-lg), 0 0 20px rgba(99, 102, 241, 0.2);
  }

  .theme-fab:active {
    transform: scale(0.95);
  }

  .theme-icon {
    width: 24px;
    height: 24px;
    position: absolute;
    opacity: 1;
    transition: opacity 200ms;
  }

  .theme-icon.hidden {
    opacity: 0;
    pointer-events: none;
  }

  @media (prefers-color-scheme: dark) {
    .theme-fab {
      background: var(--white);
      color: var(--text);
    }
  }
```

**Step 2: Commit**

```bash
git add src/public/index.html
git commit -m "feat: add FAB button styling"
```

---

### Task 5: Add Theme Detection and Toggle JavaScript

**File:** `src/public/index.html`
**Modify:** End of body, after FAB button HTML (before closing `</body>`)

Add the theme detection and toggle script. This should go right after the theme-toggle-fab button HTML.

**Step 1: Add theme script**

Right after the closing `</button>` tag for the FAB, add:

```html
  <script>
    // Theme management
    const html = document.documentElement;
    const fab = document.getElementById('theme-toggle-fab');
    const sunIcon = document.querySelector('.sun-icon');
    const moonIcon = document.querySelector('.moon-icon');
    const systemIcon = document.querySelector('.system-icon');

    // Theme cycle order: auto → light → dark → auto
    const themeCycle = ['auto', 'light', 'dark'];
    let currentThemeIndex = 0;

    // Initialize theme from localStorage or system preference
    function initTheme() {
      const stored = localStorage.getItem('claude-spend-theme');

      if (stored) {
        // User has manually set a preference
        setTheme(stored);
        currentThemeIndex = themeCycle.indexOf(stored);
      } else {
        // Follow system preference
        setTheme('auto');
        currentThemeIndex = 0;
      }
    }

    // Apply theme to DOM and update icon
    function setTheme(theme) {
      if (theme === 'auto') {
        // Remove data-theme to use default/system preference
        html.removeAttribute('data-theme');
        localStorage.removeItem('claude-spend-theme');
        showIcon('system');
      } else if (theme === 'light') {
        html.setAttribute('data-theme', 'light');
        localStorage.setItem('claude-spend-theme', 'light');
        showIcon('sun');
      } else if (theme === 'dark') {
        html.setAttribute('data-theme', 'dark');
        localStorage.setItem('claude-spend-theme', 'dark');
        showIcon('moon');
      }
    }

    // Show only the active icon
    function showIcon(icon) {
      sunIcon.classList.toggle('hidden', icon !== 'sun');
      moonIcon.classList.toggle('hidden', icon !== 'moon');
      systemIcon.classList.toggle('hidden', icon !== 'system');
    }

    // Toggle to next theme in cycle
    fab.addEventListener('click', () => {
      currentThemeIndex = (currentThemeIndex + 1) % themeCycle.length;
      setTheme(themeCycle[currentThemeIndex]);
    });

    // Apply system preference changes in real-time
    window.matchMedia('(prefers-color-scheme: dark)').addEventListener('change', (e) => {
      if (currentThemeIndex === 0) { // Only if in 'auto' mode
        // Theme will update automatically via media query
        showIcon('system');
      }
    });

    // Initialize on page load
    initTheme();
  </script>
```

**Step 2: Commit**

```bash
git add src/public/index.html
git commit -m "feat: add theme detection and toggle script"
```

---

### Task 6: Manual Visual Testing in Browser

**File:** `src/public/index.html` (view in browser)

**Step 1: Start the dev server**

```bash
npm start
```

Expected output:
```
Server running on http://localhost:3456
Opening browser...
```

**Step 2: Test theme toggle FAB**

1. FAB should appear in bottom-right corner with sun icon visible
2. Click the FAB — icon should change to moon, page should darken smoothly
3. Click again — icon should change to system icon, page should follow your OS theme preference
4. Click again — icon should change back to sun, page should be light
5. Cycle 2-3 more times to verify smooth transitions

**Step 3: Test system preference sync**

1. Set FAB to "system" mode (system icon visible)
2. Change your OS theme preference in system settings
3. Page should update automatically without clicking FAB

**Step 4: Test localStorage persistence**

1. Click FAB to set theme to "dark" (moon icon)
2. Refresh the browser (Cmd+R)
3. Page should load in dark mode immediately without flashing light theme
4. Check browser DevTools → Application → localStorage → verify `claude-spend-theme: dark` exists
5. Set to "light" mode and refresh — should persist light mode
6. Set to "system" mode and refresh — localStorage entry should be removed, page follows OS preference

**Step 5: Test readability and contrast**

1. In dark mode, verify all text is readable (7+ WCAG contrast)
2. Check that all accent colors (buttons, links, badges) are visible and distinct
3. Verify charts/graphs render correctly with dark background
4. Check that the mesh gradient background is subtle and doesn't interfere with content

**Step 6: Verify no console errors**

1. Open DevTools (F12)
2. Check Console tab — should have no errors, only normal page logs
3. Check for any visual glitches or layout shifts during theme transitions

**Step 7: Final commit**

```bash
git add -A
git commit -m "feat: complete dark theme implementation with system detection"
```

---

## Summary

This plan adds dark theme support in 6 focused tasks:
1. Dark CSS variables
2. Smooth transitions
3. FAB button HTML
4. FAB styling
5. Theme toggle script
6. Visual testing

Total changes: ~200 lines of CSS/HTML/JS in a single file. No dependencies added, no tests needed (visual validation in browser).

The implementation respects system preferences by default, allows manual override, persists user choice, and provides smooth visual feedback.
