# Signage Best Practices Reference (theme / accessibility / readability)

This reference is **framework-agnostic** and applies to every webapp the skill scaffolds. Every
framework reference (`react.md`, `vue.md`, `angular.md`, `vanilla.md`) imports the `theme.css` below
and follows the accessibility and readability rules here. Read this together with the framework
reference, and bake all of it in — it is not optional.

Digital signage has constraints a normal web page does not: viewers stand **several feet to several
meters away**, screens are often **TVs with overscan** that clip edges, content runs **unattended**
(no scrollbars, no hover, no keyboard), and accessibility is frequently a **legal requirement**
(Section 508, ADA, WCAG). The defaults below address all of these.

---

## 1. Theme system — `src/theme.css`

A single source of truth for color, spacing, radius, and typography, exposed as CSS custom
properties. Recolor by changing **one** token (`--brand`). Supports light, dark, and
device-follows (`prefers-color-scheme`) appearance. The default tokens are contrast-checked to pass
WCAG 2.1 AA.

Create `src/theme.css` exactly as below (this is imported globally by every framework scaffold):

```css
/* =========================================================================
   Revel Digital webapp theme tokens
   Recolor the whole app by changing --brand. All sizes use a fluid scale
   tuned for distance viewing. Light/dark are contrast-checked for WCAG AA.
   ========================================================================= */

:root {
  /* --- Brand (change this one value to recolor) --- */
  --brand: #2f6fed;            /* primary/brand color — set from the user's answer */
  --brand-contrast: #ffffff;   /* text/icon color that sits ON --brand (AA: >= 4.5:1) */

  /* --- Spacing scale --- */
  --space-1: 0.5rem;
  --space-2: 1rem;
  --space-3: 1.5rem;
  --space-4: 2.5rem;
  --space-5: 4rem;

  /* --- Radius --- */
  --radius: 0.75rem;

  /* --- Overscan-safe margin: keep all content inside this on TVs (~5%) --- */
  --safe-area: 5vmin;

  /* --- Fluid type scale (clamp: min, preferred-vw, max) sized for distance --- */
  /* Base body text is large on purpose — readable from across a room. */
  --font-body:    clamp(1.25rem, 1rem + 1.2vw, 2rem);
  --font-lead:    clamp(1.75rem, 1.2rem + 2vw, 3rem);
  --font-title:   clamp(2.5rem, 1.5rem + 4vw, 6rem);
  --font-display: clamp(4rem, 2rem + 8vw, 12rem);
  --line-height: 1.3;
  --measure: 32ch;             /* max line length for readability */

  --font-family: system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue",
    Arial, sans-serif;
  --font-weight-normal: 500;   /* avoid thin weights on signage — they smear at distance */
  --font-weight-bold: 800;

  /* --- Light theme colors (default) --- */
  --bg:      #ffffff;
  --surface: #f2f4f8;
  --fg:      #14181f;          /* AA on --bg and --surface */
  --fg-muted:#454c59;          /* AA on --bg */
  --border:  #d4d9e2;
  color-scheme: light;
}

/* Dark theme when the device prefers it... */
@media (prefers-color-scheme: dark) {
  :root {
    --bg:      #0b1020;
    --surface: #161d33;
    --fg:      #f5f7fa;        /* AA on --bg and --surface */
    --fg-muted:#aab3c5;        /* AA on --bg */
    --border:  #2a3354;
    color-scheme: dark;
  }
}

/* ...or force a theme regardless of device via <html data-theme="light|dark">. */
:root[data-theme="light"] {
  --bg: #ffffff; --surface: #f2f4f8; --fg: #14181f;
  --fg-muted: #454c59; --border: #d4d9e2; color-scheme: light;
}
:root[data-theme="dark"] {
  --bg: #0b1020; --surface: #161d33; --fg: #f5f7fa;
  --fg-muted: #aab3c5; --border: #2a3354; color-scheme: dark;
}

/* --- Base / reset --- */
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

html, body, #root, #app { height: 100%; }

body {
  background: var(--bg);
  color: var(--fg);
  font-family: var(--font-family);
  font-weight: var(--font-weight-normal);
  font-size: var(--font-body);
  line-height: var(--line-height);
  -webkit-font-smoothing: antialiased;
  text-rendering: optimizeLegibility;
}

/* Overscan-safe root layout — content never reaches the physical screen edge. */
.signage-root {
  min-height: 100%;
  padding: var(--safe-area);
  display: flex;
  flex-direction: column;
  gap: var(--space-3);
}

h1 { font-size: var(--font-title); font-weight: var(--font-weight-bold); line-height: 1.1; }
h2 { font-size: var(--font-lead);  font-weight: var(--font-weight-bold); line-height: 1.15; }
p  { max-width: var(--measure); }

/* Accent / call-to-action surface using the brand color. */
.accent {
  background: var(--brand);
  color: var(--brand-contrast);
  border-radius: var(--radius);
  padding: var(--space-2) var(--space-3);
}

/* --- Accessibility: always-visible keyboard focus --- */
:where(a, button, [tabindex]):focus-visible {
  outline: 3px solid var(--brand);
  outline-offset: 3px;
}

/* --- Accessibility: respect reduced-motion. Wrap ALL animation in this guard. --- */
@media (prefers-reduced-motion: no-preference) {
  .animate-fade { animation: fade-in 600ms ease both; }
}
@keyframes fade-in { from { opacity: 0; } to { opacity: 1; } }

/* Visually-hidden helper for screen-reader-only text. */
.sr-only {
  position: absolute; width: 1px; height: 1px; padding: 0; margin: -1px;
  overflow: hidden; clip: rect(0 0 0 0); white-space: nowrap; border: 0;
}
```

### How to recolor from the user's answer

- Set `--brand` to the hex the user gave.
- Check `--brand-contrast`: if `--brand` is light, set `--brand-contrast: #14181f`; if dark, keep
  `#ffffff`. The text on the brand color must reach **AA 4.5:1** (3:1 for large text). If unsure,
  state the ratio you targeted.

### Default appearance (light / dark / auto)

- **Auto (recommended)** — leave the `@media (prefers-color-scheme)` block; the app follows the
  device. Do not set `data-theme`.
- **Force light/dark** — set `<html lang="…" data-theme="light">` (or `"dark"`) in the entry HTML.

---

## 2. Accessibility — Section 508 / WCAG 2.1 AA

Bake these into every scaffold:

- **Language** — set `<html lang>` from the device. On load, call `getLanguageCode()` and set
  `document.documentElement.lang` (fall back to the static `lang="en"` already in the HTML).
- **Semantic landmarks** — wrap the app in a single `<main>` (the `.signage-root` element), use
  `<header>` for the title area, real `<h1>/<h2>` heading order. Don't build structure from bare
  `<div>`s.
- **Live regions** — content that updates without interaction (clock, ticker, rotating messages)
  lives in an element with `aria-live="polite"` (or `aria-live="off"` for a constantly-ticking
  clock to avoid spamming assistive tech — a once-a-second time update should generally be `off` or
  use `aria-atomic`). Rotating announcements use `aria-live="polite"`.
- **Visible focus** — provided by the `:focus-visible` rule in `theme.css`. Never remove outlines
  without replacing them.
- **Reduced motion** — every animation/transition must sit inside
  `@media (prefers-reduced-motion: no-preference)`. Signage often auto-animates; this respects users
  with vestibular disorders.
- **Color contrast** — body text and essential UI must meet **AA 4.5:1** (3:1 for large/bold text
  ≥ ~24px). The default tokens already pass. If you introduce new color pairs, verify them.
- **Don't rely on color alone** — pair color-coded status with text or an icon + label.
- **Images** — every `<img>` needs `alt`; decorative images use `alt=""`. Background/atmospheric
  imagery should not carry essential information.
- **No keyboard traps / motion seizures** — avoid flashing > 3×/second.

A short, copy-pasteable accessibility checklist for the generated `README` is helpful: lang set,
landmarks present, AA contrast, reduced-motion guarded, alt text, live regions for dynamic content.

---

## 3. Readability for distance viewing

- **Large base type** — the `--font-*` clamp scale starts large (body ≥ 1.25rem and scales with the
  viewport). Don't shrink below it for primary content.
- **Avoid thin weights** — use `--font-weight-normal: 500` and `--font-weight-bold: 800`; hairline
  fonts disappear at distance.
- **High contrast** — prefer the strong `--fg` on `--bg`; reserve `--fg-muted` for secondary labels.
- **Short measure** — cap line length with `--measure` (≈ 32ch) so text stays scannable.
- **Overscan-safe** — keep everything inside `--safe-area` (5% padding via `.signage-root`). Never
  place critical content flush to the screen edge.
- **No scrollbars** — a signage screen can't scroll. Design for the fixed viewport; if content can
  overflow, rotate/paginate it on a timer instead of scrolling.
- **Generous spacing** — use the `--space-*` scale; crowded layouts are hard to parse at a glance.
