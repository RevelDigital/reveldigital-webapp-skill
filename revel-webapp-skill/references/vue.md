# Vue Webapp Reference

Scaffold a full-screen Revel Digital **webapp** using Vite + Vue 3, wired to
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
│   ├── App.vue
│   ├── App.css
│   ├── main.js
│   ├── theme.css               # from references/signage.md (verbatim)
│   ├── config.js               # config.json loader
│   └── playerClient.js         # SDK composable (device data, clock, commands)
├── .github/
│   └── workflows/
│       └── deploy.yml          # only if deploying (from references/deploy.md)
├── index.html
├── package.json
└── vite.config.js
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
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "@reveldigital/client-sdk": "latest",
    "vue": "^3.5.0"
  },
  "devDependencies": {
    "@vitejs/plugin-vue": "^5.2.0",
    "vite": "^6.3.5"
  }
}
```

## vite.config.js

```javascript
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  base: './',          // relative asset paths so the player can serve from anywhere
  build: {
    outDir: 'dist',    // matches webapp-action's default distribution-location
  },
})
```

## index.html

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>{app-name}</title>
  </head>
  <body>
    <div id="app"></div>
    <script type="module" src="/src/main.js"></script>
  </body>
</html>
```

Omit `data-theme` to follow the device (`prefers-color-scheme`); set `data-theme="light"` or
`"dark"` on `<html>` to force the user's chosen appearance.

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

## src/config.js

```javascript
const DEFAULT_CONFIG = {
  title: 'Welcome',
  messages: ['Configure public/config.json to set your content.'],
  rotateSeconds: 8,
  showClock: true,
  showLocation: true,
}

export async function loadConfig() {
  try {
    const res = await fetch(`${import.meta.env.BASE_URL}config.json`, { cache: 'no-store' })
    if (!res.ok) return DEFAULT_CONFIG
    return { ...DEFAULT_CONFIG, ...(await res.json()) }
  } catch {
    return DEFAULT_CONFIG
  }
}
```

## src/playerClient.js

A composable that wires the SDK: live device clock, device/location context, command handling, and
graceful fallbacks off-device. The command listener is cleaned up on unmount.

```javascript
import { ref, onMounted, onUnmounted } from 'vue'
import { createPlayerClient, EventType } from '@reveldigital/client-sdk'

export function usePlayerClient() {
  const time = ref(null)
  const timeZone = ref(null)   // device timezone (IANA) — format the clock with this
  const device = ref(null)
  const isPreview = ref(false)
  const lastCommand = ref(null)

  let client = null
  let clockId = null
  let resyncId = null
  let offsetMs = 0             // deviceTime - localTime
  const onCommand = (data) => {
    lastCommand.value = `${data?.name ?? 'command'}${data?.arg ? `: ${data.arg}` : ''}`
  }

  onMounted(() => {
    client = createPlayerClient()

    client.getDevice().then((d) => { device.value = d }).catch(() => {})
    client.isPreviewMode().then((p) => { isPreview.value = Boolean(p) }).catch(() => {})

    // Render the clock in the DEVICE's timezone, not the browser's.
    client.getDeviceTimeZoneName().then((tz) => { if (tz) timeZone.value = tz }).catch(() => {})

    // Device clock: derive the device↔local offset from getDeviceTime() once, then tick
    // locally — accurate device time without calling the player shim every second.
    const syncOffset = async () => {
      try {
        const iso = await client.getDeviceTime()
        if (iso) offsetMs = new Date(iso).getTime() - Date.now()
      } catch { offsetMs = 0 }   // off-device → local time
    }
    syncOffset().then(() => { time.value = new Date(Date.now() + offsetMs) })
    clockId = window.setInterval(() => { time.value = new Date(Date.now() + offsetMs) }, 1000)
    resyncId = window.setInterval(syncOffset, 5 * 60 * 1000)

    try { client.on(EventType.COMMAND, onCommand) } catch { /* off-device */ }

    client.getLanguageCode()
      .then((lang) => { if (lang) document.documentElement.lang = lang })
      .catch(() => {})
  })

  onUnmounted(() => {
    if (clockId) window.clearInterval(clockId)
    if (resyncId) window.clearInterval(resyncId)
    try { client?.off(EventType.COMMAND) } catch { /* noop */ }
  })

  return { time, timeZone, device, isPreview, lastCommand }
}
```

## src/main.js

```javascript
import { createApp } from 'vue'
import './theme.css'
import './App.css'
import App from './App.vue'

createApp(App).mount('#app')
```

## src/App.vue

```vue
<script setup>
import { ref, computed, onMounted, onUnmounted, watch } from 'vue'
import { loadConfig } from './config'
import { usePlayerClient } from './playerClient'

const config = ref(null)
const msgIndex = ref(0)
const { time, timeZone, device, isPreview, lastCommand } = usePlayerClient()

let rotateId = null

onMounted(async () => {
  config.value = await loadConfig()
})

watch(config, (cfg) => {
  if (rotateId) window.clearInterval(rotateId)
  if (cfg && cfg.messages.length > 1) {
    const ms = Math.max(2, cfg.rotateSeconds) * 1000
    rotateId = window.setInterval(() => {
      msgIndex.value = (msgIndex.value + 1) % cfg.messages.length
    }, ms)
  }
})

onUnmounted(() => { if (rotateId) window.clearInterval(rotateId) })

const clock = computed(() =>
  time.value
    ? new Intl.DateTimeFormat(undefined, {
        hour: '2-digit', minute: '2-digit', second: '2-digit',
        timeZone: timeZone.value ?? undefined,   // render in the device's timezone
      }).format(time.value)
    : '--:--',
)

const place = computed(() =>
  device.value?.location
    ? [device.value.location.city, device.value.location.state].filter(Boolean).join(', ')
    : null,
)
</script>

<template>
  <main class="signage-root">
    <template v-if="config">
      <header class="header">
        <h1>{{ config.title }}</h1>
        <span v-if="isPreview" class="badge">Preview</span>
      </header>

      <section v-if="config.showClock" class="clock" aria-live="off" aria-label="Current time">
        {{ clock }}
      </section>

      <section
        class="message accent animate-fade"
        aria-live="polite"
        :key="msgIndex"
      >
        <p>{{ config.messages[msgIndex] }}</p>
      </section>

      <footer class="footer">
        <span v-if="config.showLocation && place" class="footer-item">📍 {{ place }}</span>
        <span v-if="device?.name" class="footer-item">{{ device.name }}</span>
        <span v-if="lastCommand" class="footer-item">Last command — {{ lastCommand }}</span>
      </footer>
    </template>
    <h1 v-else>Loading…</h1>
  </main>
</template>
```

## src/App.css

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
