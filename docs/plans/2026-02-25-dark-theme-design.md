# Dark Theme Design Document

**Date:** 2026-02-25
**Feature:** Dark Mode with System Preference Detection
**Status:** Approved

## Overview

Add a dark theme to the Claude Spend dashboard with a floating action button (FAB) toggle. The theme will follow the user's system preference (light/dark) by default, with manual override capability. The dark palette uses neutral blacks/grays with bright accent colors maintained from the light theme.

## Design Decisions

### 1. Theme Toggle UI
- **Location:** Floating action button in bottom-right corner
- **Icon cycling:** Sun (light) → Moon (dark) → System icon (auto) → repeats
- **Interaction:** Single click to cycle through modes
- **Visual style:** Semi-transparent background, subtle glow, smooth icon transitions
- **Positioning:** Fixed, 24px from bottom/right edges, above other content

### 2. Detection & Storage
- **Default behavior:** Auto-detect via `prefers-color-scheme` CSS media query
- **Storage:** localStorage key `claude-spend-theme`
  - Values: `"light"` | `"dark"` | `null` (null = follow system)
  - Only stored if user manually overrides system preference
- **Initial load:** Check localStorage → if null, check system preference → apply theme

### 3. Color Palette

#### Light Theme (Current)
```
Background: #F8F9FC
Mesh: #EEF0FF, #F0FDFA
Text: #0F172A
Text Secondary: #475569
Text Tertiary: #94A3B8
Border: rgba(0,0,0,0.06)
Border Strong: rgba(0,0,0,0.1)
```

#### Dark Theme (New)
```
Background: #0F172A (dark slate)
Mesh Backgrounds: #1A1F35, #1A2B2A (subtle blues/teals)
Text: #F1F5F9 (light gray)
Text Secondary: #CBD5E1 (medium gray)
Text Tertiary: #64748B (muted gray)
Border: rgba(255,255,255,0.08)
Border Strong: rgba(255,255,255,0.12)
```

**Accent colors:** Maintained from light theme (indigo, violet, purple, teal, cyan, emerald, amber, orange, rose, blue) — these are bright enough to stand out on dark backgrounds.

**Gradients:** Slightly desaturated in dark mode for reduced eye strain while maintaining visual hierarchy.

**Shadows:** Adjusted — dark theme uses lighter shadow colors (rgba(255,255,255,0.06) range) to create depth without appearing washed out.

### 4. Implementation Strategy

1. **CSS layer:** Add `:root[data-theme="dark"]` rule set with dark palette variables
2. **JavaScript:** Small script (< 50 lines) to handle:
   - System preference detection on load
   - Toggle logic and icon cycling
   - localStorage read/write
3. **DOM:** Single FAB button with SVG icons (sun, moon, system)
4. **Transitions:** `transition: background-color 300ms, color 300ms` on body and key elements for smooth theme switching

### 5. Success Criteria

- ✓ Dark theme is readable with 7+ WCAG contrast ratio on all text
- ✓ Accent colors remain vibrant and distinguishable in dark mode
- ✓ System preference is respected on first load
- ✓ Manual toggle overrides system preference and persists
- ✓ Clicking FAB cycles: auto → light → dark → auto
- ✓ Smooth 300ms color transitions when toggling
- ✓ No layout shifts or flashing when theme loads

### 6. Files Modified

- **`src/public/index.html`**
  - Add dark palette CSS variables under `:root[data-theme="dark"]`
  - Add FAB button HTML and styling
  - Add theme detection + toggle script
  - Add `transition` properties to elements with color changes

### 7. No New Dependencies

The implementation uses only native CSS and vanilla JS — no additional npm packages required.

## Next Steps

Create implementation plan with specific line-by-line changes to index.html.
