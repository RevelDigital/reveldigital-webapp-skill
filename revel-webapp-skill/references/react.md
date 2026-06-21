# React Webapp Reference

Scaffold a full-screen Revel Digital **webapp** using Vite + React + TypeScript, wired to
`@reveldigital/client-sdk`. Read `references/signage.md` and `references/deploy.md` alongside this
file — `theme.css`, the accessibility rules, and the deploy workflow come from there.

This is a **webapp**, not a gadget: no `gadget.yaml`, no gadgetizer, no GitHub Pages. The build
output is a static `dist/` that the CMS deploy action uploads.

## Project Structure

```
{app-name}/
├── public/
│   └── config.json             # runtime content (editable without rebuilding)
├── src/
│   ├── App.tsx
│   ├── App.css
│   ├── main.tsx
│   ├── theme.css               # from references/signage.md (verbatim)
│   ├── config.ts               # config.json loader + types
│   ├── usePlayerClient.ts      # SDK hook (device data, clock, commands)
│   └── vite-env.d.ts
├── .github/
│   └── workflows/
│       └── deploy.yml          # only if deploying (from references/deploy.md)
├── index.html
├── package.json
├── tsconfig.json
├── tsconfig.app.json
├── tsconfig.node.json
└── vite.config.ts
```

## package.json

```json
{
  "name": "{app-name}",
  "private": true,
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "@reveldigital/client-sdk": "latest",
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  },
  "devDependencies": {
    "@types/react": "^19.0.0",
    "@types/react-dom": "^19.0.0",
    "@vitejs/plugin-react": "^4.4.1",
    "typescript": "~5.7.2",
    "vite": "^6.3.5"
  }
}
```

## vite.config.ts

```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  base: './',          // relative asset paths so the player can serve from anywhere
  build: {
    outDir: 'dist',    // matches webapp-action's default distribution-location
  },
})
```

## tsconfig.json

```json
{
  "files": [],
  "references": [
    { "path": "./tsconfig.app.json" },
    { "path": "./tsconfig.node.json" }
  ]
}
```

## tsconfig.app.json

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
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "resolveJsonModule": true
  },
  "include": ["src"]
}
```

Note: `isolatedModules` is intentionally omitted — the SDK's `EventType` is a `const enum`, which is
incompatible with `isolatedModules`.

## tsconfig.node.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2023"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "moduleDetection": "force",
    "noEmit": true,
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true
  },
  "include": ["vite.config.ts"]
}
```

## src/vite-env.d.ts

```typescript
/// <reference types="vite/client" />
```

## index.html

`lang` is set statically and then refined at runtime from the device language. `data-theme` is
omitted so the app follows the device (`prefers-color-scheme`); set `data-theme="light"` or
`"dark"` here to force an appearance per the user's choice.

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>{app-name}</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

## public/config.json

Webapps have no `gadget.yaml` preferences UI, so runtime content is driven by this file. It lives in
`public/`, so it is copied to `dist/config.json` and can be edited after the build without
recompiling.

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

// Loaded relative to the deployed app so it works wherever the player serves from.
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

## src/usePlayerClient.ts

A small hook that wires the SDK: a live device clock, device/location context, command handling, and
graceful fallbacks off-device (so `npm run dev` works in a plain browser). Every async SDK call is
wrapped in `.catch()`, and the command listener is cleaned up on unmount.

```typescript
import { useEffect, useState } from 'react'
import { createPlayerClient, EventType } from '@reveldigital/client-sdk'
import type { IDevice } from '@reveldigital/client-sdk'

export interface PlayerState {
  time: Date | null
  timeZone: string | null   // device timezone (IANA) — format the clock with this
  device: IDevice | null
  isPreview: boolean
  lastCommand: string | null
}

export function usePlayerClient(): PlayerState {
  const [device, setDevice] = useState<IDevice | null>(null)
  const [isPreview, setIsPreview] = useState(false)
  const [time, setTime] = useState<Date | null>(null)
  const [timeZone, setTimeZone] = useState<string | null>(null)
  const [lastCommand, setLastCommand] = useState<string | null>(null)

  useEffect(() => {
    const client = createPlayerClient()
    let active = true

    // One-time device + preview context.
    client.getDevice().then((d) => active && setDevice(d as IDevice | null)).catch(() => {})
    client.isPreviewMode().then((p) => active && setIsPreview(Boolean(p))).catch(() => {})

    // Render the clock in the DEVICE's timezone (getDeviceTimeZoneName), not the browser's.
    client.getDeviceTimeZoneName().then((tz) => { if (tz && active) setTimeZone(tz) }).catch(() => {})

    // Device clock: use getDeviceTime() to derive the device↔local offset once, then tick
    // locally — accurate device time without calling the player shim every second.
    // Re-sync periodically to correct drift. Off-device, offset is 0 (local time).
    let offsetMs = 0
    const syncOffset = async () => {
      try {
        const iso = await client.getDeviceTime()        // device time, ISO8601
        if (iso) offsetMs = new Date(iso).getTime() - Date.now()
      } catch { offsetMs = 0 }
    }
    syncOffset().then(() => { if (active) setTime(new Date(Date.now() + offsetMs)) })
    const clockId = window.setInterval(() => {
      if (active) setTime(new Date(Date.now() + offsetMs))
    }, 1000)
    const resyncId = window.setInterval(syncOffset, 5 * 60 * 1000)

    // Command handling — react to player commands.
    const onCommand = (data: { name?: string; arg?: string }) => {
      setLastCommand(`${data?.name ?? 'command'}${data?.arg ? `: ${data.arg}` : ''}`)
    }
    try { client.on(EventType.COMMAND, onCommand) } catch { /* off-device */ }

    // Set <html lang> from the device language (getLanguageCode), not navigator.language.
    client.getLanguageCode()
      .then((lang) => { if (lang) document.documentElement.lang = lang })
      .catch(() => {})

    return () => {
      active = false
      window.clearInterval(clockId)
      window.clearInterval(resyncId)
      try { client.off(EventType.COMMAND) } catch { /* noop */ }
    }
  }, [])

  return { time, timeZone, device, isPreview, lastCommand }
}
```

## src/main.tsx

```tsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import './theme.css'
import './App.css'
import App from './App'

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <App />
  </StrictMode>,
)
```

## src/App.tsx

A signage-ready layout: a full-screen, overscan-safe `<main>` with a header title, a large live
clock, device location, and rotating announcements in an `aria-live` region — all driven by
`config.json`.

```tsx
import { useEffect, useState } from 'react'
import { loadConfig, type AppConfig } from './config'
import { usePlayerClient } from './usePlayerClient'

function App() {
  const [config, setConfig] = useState<AppConfig | null>(null)
  const [msgIndex, setMsgIndex] = useState(0)
  const { time, timeZone, device, isPreview, lastCommand } = usePlayerClient()

  useEffect(() => { loadConfig().then(setConfig) }, [])

  // Rotate announcements on the configured interval.
  useEffect(() => {
    if (!config || config.messages.length <= 1) return
    const ms = Math.max(2, config.rotateSeconds) * 1000
    const id = window.setInterval(
      () => setMsgIndex((i) => (i + 1) % config.messages.length),
      ms,
    )
    return () => window.clearInterval(id)
  }, [config])

  if (!config) {
    return <main className="signage-root"><h1>Loading…</h1></main>
  }

  const clock = time
    ? new Intl.DateTimeFormat(undefined, {
        hour: '2-digit', minute: '2-digit', second: '2-digit',
        timeZone: timeZone ?? undefined,   // render in the device's timezone
      }).format(time)
    : '--:--'

  const place = device?.location
    ? [device.location.city, device.location.state].filter(Boolean).join(', ')
    : null

  return (
    <main className="signage-root">
      <header className="header">
        <h1>{config.title}</h1>
        {isPreview && <span className="badge">Preview</span>}
      </header>

      {config.showClock && (
        <section className="clock" aria-live="off" aria-label="Current time">
          {clock}
        </section>
      )}

      <section className="message accent animate-fade" aria-live="polite" key={msgIndex}>
        <p>{config.messages[msgIndex]}</p>
      </section>

      <footer className="footer">
        {config.showLocation && place && (
          <span className="footer-item">📍 {place}</span>
        )}
        {device?.name && <span className="footer-item">{device.name}</span>}
        {lastCommand && (
          <span className="footer-item">Last command — {lastCommand}</span>
        )}
      </footer>
    </main>
  )
}

export default App
```

## src/App.css

App-specific layout on top of `theme.css`. Keep all spacing on the token scale and animations inside
the reduced-motion guard (provided by `theme.css`).

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
npm install     # install dependencies
npm run dev      # develop in the browser — SDK calls fall back gracefully off-device
npm run build    # production build to ./dist (contains index.html + config.json)
npm run preview  # serve the production build locally
```

The `dist/` folder is exactly what `RevelDigital/webapp-action` uploads to the CMS. See
`references/deploy.md` for the deploy workflow.
