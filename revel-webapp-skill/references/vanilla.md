# Vanilla JS Webapp Reference

Scaffold a full-screen Revel Digital **webapp** using Vite + TypeScript with no UI framework, wired
to `@reveldigital/client-sdk`. Read `references/signage.md` and `references/deploy.md` alongside this
file — `theme.css`, the accessibility rules, and the deploy workflow come from there.

This is a **webapp**, not a gadget: no `gadget.yaml`, no gadgetizer, no GitHub Pages. The build
output is a static `dist/` that the CMS deploy action uploads.

## Project Structure

```
{app-name}/
├── public/
│   └── config.json             # runtime content (editable without rebuilding)
├── src/
│   ├── main.ts                 # app logic + SDK wiring
│   ├── theme.css               # from references/signage.md (verbatim)
│   ├── app.css
│   ├── config.ts               # config.json loader + types
│   └── vite-env.d.ts
├── .github/
│   └── workflows/
│       └── deploy.yml          # only if deploying (from references/deploy.md)
├── index.html
├── package.json
├── tsconfig.json
└── vite.config.ts
```

> Copy `src/theme.css` **verbatim** from `references/signage.md` (section 1).

## package.json

```json
{
  "name": "{app-name}",
  "private": true,
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "@reveldigital/client-sdk": "latest"
  },
  "devDependencies": {
    "typescript": "~5.7.2",
    "vite": "^6.3.5"
  }
}
```

## vite.config.ts

```typescript
import { defineConfig } from 'vite'

export default defineConfig({
  base: './',          // relative asset paths so the player can serve from anywhere
  build: {
    outDir: 'dist',    // matches webapp-action's default distribution-location
  },
})
```

## tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "moduleDetection": "force",
    "noEmit": true,
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "resolveJsonModule": true
  },
  "include": ["src"]
}
```

Note: `isolatedModules` is intentionally omitted — the SDK's `EventType` is a `const enum`.

## src/vite-env.d.ts

```typescript
/// <reference types="vite/client" />
```

## index.html

Semantic landmarks are in the markup; `main.ts` fills in the dynamic text. Omit `data-theme` to
follow the device, or set `data-theme="light"`/`"dark"` to force the user's chosen appearance.

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>{app-name}</title>
  </head>
  <body>
    <main class="signage-root">
      <header class="header">
        <h1 id="title">Welcome</h1>
        <span id="badge" class="badge" hidden>Preview</span>
      </header>

      <section id="clock" class="clock" aria-live="off" aria-label="Current time">--:--</section>

      <section id="message" class="message accent" aria-live="polite">
        <p id="message-text">Loading…</p>
      </section>

      <footer class="footer">
        <span id="place" class="footer-item" hidden></span>
        <span id="device-name" class="footer-item" hidden></span>
        <span id="last-command" class="footer-item" hidden></span>
      </footer>
    </main>
    <script type="module" src="/src/main.ts"></script>
  </body>
</html>
```

## public/config.json

```json
{
  "title": "Welcome",
  "messages": [
    "Edit public/config.json to change this content.",
    "No rebuild required — just replace config.json.",
    "Designed for accessibility and distance readability."
  ],
  "rotateSeconds": 8,
  "showClock": true,
  "showLocation": true
}
```

## src/config.ts

```typescript
export interface AppConfig {
  title: string
  messages: string[]
  rotateSeconds: number
  showClock: boolean
  showLocation: boolean
}

const DEFAULT_CONFIG: AppConfig = {
  title: 'Welcome',
  messages: ['Configure public/config.json to set your content.'],
  rotateSeconds: 8,
  showClock: true,
  showLocation: true,
}

export async function loadConfig(): Promise<AppConfig> {
  try {
    const res = await fetch(`${import.meta.env.BASE_URL}config.json`, { cache: 'no-store' })
    if (!res.ok) return DEFAULT_CONFIG
    return { ...DEFAULT_CONFIG, ...(await res.json()) }
  } catch {
    return DEFAULT_CONFIG
  }
}
```

## src/main.ts

```typescript
import './theme.css'
import './app.css'
import { createPlayerClient, EventType } from '@reveldigital/client-sdk'
import { loadConfig } from './config'

const $ = (id: string) => document.getElementById(id)!

function show(id: string, text: string) {
  const el = $(id)
  el.textContent = text
  el.hidden = false
}

async function main() {
  const config = await loadConfig()
  $('title').textContent = config.title
  if (!config.showClock) $('clock').hidden = true

  // Rotate announcements.
  let msgIndex = 0
  const render = () => { $('message-text').textContent = config.messages[msgIndex] }
  render()
  if (config.messages.length > 1) {
    const ms = Math.max(2, config.rotateSeconds) * 1000
    window.setInterval(() => {
      msgIndex = (msgIndex + 1) % config.messages.length
      render()
    }, ms)
  }

  const client = createPlayerClient()

  // Preview badge.
  client.isPreviewMode().then((p) => { if (p) $('badge').hidden = false }).catch(() => {})

  // <html lang> for accessibility.
  client.getLanguageCode()
    .then((lang) => { if (lang) document.documentElement.lang = lang })
    .catch(() => {})

  // Device + location context.
  client.getDevice().then((device: any) => {
    if (!device) return
    if (device.name) show('device-name', device.name)
    if (config.showLocation && device.location) {
      const place = [device.location.city, device.location.state].filter(Boolean).join(', ')
      if (place) show('place', `📍 ${place}`)
    }
  }).catch(() => {})

  // Device clock: format in the device's timezone (getDeviceTimeZoneName), and derive the
  // device↔local offset from getDeviceTime() once, then tick locally — accurate device time
  // without calling the player shim every second. Off-device, the offset is 0 (local time).
  let offsetMs = 0
  let timeZone: string | undefined
  client.getDeviceTimeZoneName().then((tz) => { if (tz) timeZone = tz }).catch(() => {})
  const syncOffset = async () => {
    try {
      const iso = await client.getDeviceTime()
      if (iso) offsetMs = new Date(iso).getTime() - Date.now()
    } catch { offsetMs = 0 }
  }
  const render = () => {
    $('clock').textContent = new Intl.DateTimeFormat(undefined, {
      hour: '2-digit', minute: '2-digit', second: '2-digit', timeZone,
    }).format(new Date(Date.now() + offsetMs))
  }
  syncOffset().then(render)
  window.setInterval(render, 1000)
  window.setInterval(syncOffset, 5 * 60 * 1000)

  // Command handling.
  try {
    client.on(EventType.COMMAND, (data: { name?: string; arg?: string }) => {
      show('last-command', `Last command — ${data?.name ?? 'command'}${data?.arg ? `: ${data.arg}` : ''}`)
    })
  } catch { /* off-device */ }
}

main()
```

## src/app.css

```css
.header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: var(--space-2);
}

.badge {
  font-size: var(--font-body);
  font-weight: var(--font-weight-bold);
  color: var(--brand-contrast);
  background: var(--brand);
  border-radius: var(--radius);
  padding: var(--space-1) var(--space-2);
}

.clock {
  font-size: var(--font-display);
  font-weight: var(--font-weight-bold);
  font-variant-numeric: tabular-nums;
  line-height: 1;
  text-align: center;
}

.message {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  text-align: center;
}
.message p {
  max-width: 24ch;
  font-size: var(--font-lead);
}

.footer {
  display: flex;
  flex-wrap: wrap;
  gap: var(--space-3);
  justify-content: center;
  color: var(--fg-muted);
  font-size: var(--font-body);
}
```

## Build & Development

```bash
npm install
npm run dev      # develop in the browser — SDK calls fall back gracefully off-device
npm run build    # production build to ./dist (contains index.html + config.json)
npm run preview  # serve the production build locally
```

The `dist/` folder is what `RevelDigital/webapp-action` uploads to the CMS. See
`references/deploy.md`.
