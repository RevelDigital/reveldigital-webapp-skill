---
name: revel-webapp
description: Scaffold, build, and deploy full-screen Revel Digital webapps using the @reveldigital/client-sdk. Use this skill whenever the user wants to create a Revel Digital webapp, a digital signage app or screen, full-screen signage content, a kiosk or display app, or mentions deploying a web app to the Revel Digital CMS / player via the client SDK. Bakes in a theme system, Section 508 / WCAG accessibility, and distance readability. Covers React, Vue, Angular, and Vanilla JS. Can deploy straight from Claude Code via the Revel Digital MCP server (import_media) or via the RevelDigital/webapp-action GitHub Action. Use the separate revel-gadget skill instead when the user wants an embeddable gadget/widget defined by gadget.yaml and hosted on GitHub Pages.
---

# Revel Digital Webapp Skill

This skill scaffolds a complete **Revel Digital webapp** — a self-contained, full-screen HTML5
application that runs on a Revel Digital player and communicates with the player through the
`@reveldigital/client-sdk` shim. The scaffolded project is signage-ready out of the box: it ships
with a theme system, accessibility (Section 508 / WCAG 2.1 AA), distance-readable typography, and a
one-step deploy to the Revel Digital CMS.

## Webapp vs. gadget — read this first

A **webapp** and a **gadget** are different deliverables. This skill builds **webapps**.

|                  | Webapp (this skill)                              | Gadget (revel-gadget skill)                |
|------------------|--------------------------------------------------|--------------------------------------------|
| Runs as          | Full-screen app on the player                    | Embedded widget inside Revel content       |
| Definition file  | **None** — content driven by `config.json`       | `gadget.yaml`                              |
| Build output     | `dist/` (static site)                            | `dist/` + generated gadget XML             |
| Packaged as      | `<name>.webapp` (zip)                            | gadget XML descriptor                      |
| Deployed to      | **Revel Digital CMS** via `webapp-action`        | GitHub Pages via gadgetizer                |

**Do NOT** generate `gadget.yaml`, run `gadgetizer`, add a `build:gadget` script, or deploy to
GitHub Pages for a webapp. If the user actually wants a gadget, point them at the `revel-gadget`
skill instead.

## Workflow

### 1. Ask the user a few questions

Before scaffolding, gather:

- **Framework** — present the choice (React is the recommended default):
  - **React** — Vite + React + TypeScript *(default)*
  - **Vue** — Vite + Vue 3
  - **Angular** — Angular CLI + `@reveldigital/player-client` (Angular-native, DI & RxJS)
  - **Vanilla JS** — Vite + TypeScript, no framework
- **App name** — used for the project directory and `package.json` `name` (the deploy action reads
  the webapp name from here).
- **Theme basics** — a primary/brand color (hex), and whether the default appearance should be
  light, dark, or follow the device (`prefers-color-scheme`). These feed the theme tokens in
  `theme.css`.
- **How will they deploy?** — offer the options (recommend the first):
  - **Via the Revel Digital MCP server (recommended)** — build and deploy right now in the
    conversation, no CI, no repo secret, no files added to the project. Works in any MCP-capable,
    skill-capable host (Claude Code and others). Requires the MCP server connected; see step 6 and
    `references/deploy.md` (Option A).
  - **GitHub Action (CI)** — add `.github/workflows/deploy.yml` for hands-off redeploy on every
    `git push` (needs a repo + `Revel_API_Key` secret).
  - **Neither yet** — just scaffold; deploy later.

### 2. Read the reference files before generating any code

Always read **three** files: the chosen framework file **plus** `signage.md` **and** `deploy.md`.

| Framework  | Reference file          |
|------------|-------------------------|
| React      | `references/react.md`   |
| Vue        | `references/vue.md`     |
| Angular    | `references/angular.md` |
| Vanilla JS | `references/vanilla.md` |

| Always also read | Purpose                                                            |
|------------------|--------------------------------------------------------------------|
| `references/signage.md` | Theme tokens, accessibility (508/WCAG), readability — baked into every app |
| `references/deploy.md`  | CMS deploy workflow (`webapp-action`) + advisory quality report   |

Follow the reference files precisely — they contain the exact file structure, dependencies, and
sample code.

### 3. Scaffold the full project

Generate **every** file listed in the framework reference, complete (no elided snippets). The
project must `npm install` and `npm run build` cleanly to a `dist/` folder containing `index.html`
— that `dist/` folder is exactly what the deploy action uploads.

### 4. Bake in the signage best practices (not optional)

Apply `references/signage.md` to every scaffold:

- Ship `theme.css` with the design tokens and import it globally. Recolor `--brand` from the user's
  answer; set the default `data-theme` / `color-scheme` per their light/dark/auto choice.
- Use semantic landmarks (`<main>`, `<header>`), set `<html lang>` from `getLanguageCode()`, add
  visible focus styles, and guard animations with `prefers-reduced-motion`.
- Use the fluid `clamp()` type scale and the overscan-safe `--safe-area` padding on the root layout.

### 5. Wire up the Client SDK demo UX

**Use the Client SDK, not browser equivalents.** The player exposes device context through the SDK —
always prefer it over the browser API, and fall back to the browser only when off-device (wrap SDK
calls in `try`/`.catch()`):

| Need | Use (SDK) | Instead of |
|------|-----------|------------|
| Time / clock | `getDeviceTime()` + `getDeviceTimeZoneName()` (format in the device timezone) | `new Date()` rendered in the browser's timezone |
| Timezone | `getDeviceTimeZoneName()` / `getDeviceTimeZoneID()` / `getDeviceTimeZoneOffset()` | `Intl…resolvedOptions().timeZone` |
| Language / locale | `getLanguageCode()` (set `<html lang>`) | `navigator.language` |
| Location (geo) | `getDevice().location` (lat/long, city/state) | `navigator.geolocation` |
| Screen / zone size | `getWidth()` / `getHeight()` | `window.innerWidth/innerHeight` |
| Scheduled duration | `getDuration()` | — |
| Preview vs. live | `isPreviewMode()` | assuming live |
| Commands | `on(EventType.COMMAND, …)` / `sendCommand()` | — |
| Analytics | `track()` / `timeEvent()` | — |

> **Device-clock pattern (do this, don't poll the shim every second):** call `getDeviceTime()` once
> to compute the device↔local offset, tick locally each second from that offset (re-sync every few
> minutes for drift), and format with `getDeviceTimeZoneName()` so the clock shows the device's
> wall-clock time regardless of where the browser runs. Off-device, the offset is 0 → local time.
> See `usePlayerClient` in the framework references for the exact implementation.

The scaffolded app should be an immediately useful signage starting point that demonstrates the SDK:

1. Initialize the client (`createPlayerClient()`, or inject `PlayerClientService` for Angular).
2. Show a **live device clock** per the device-clock pattern above (`getDeviceTime()` offset +
   `getDeviceTimeZoneName()` formatting) — never a bare `new Date()`.
3. Show device/location context from `getDevice()` (name, city/state from `location`).
4. **Handle commands** — subscribe to `EventType.COMMAND` (React/Vue/Vanilla) or `onCommand$`
   (Angular) and react to a sample command.
5. **Respect `isPreviewMode()`** — fall back gracefully so the app renders in a plain browser during
   development (wrap every async SDK call in `.catch()`).
6. Drive display content (title, messages, rotation interval) from **`config.json`** rather than
   hard-coded values — this is the webapp equivalent of gadget preferences.

### 5b. Sourcing dynamic data (external APIs, data tables)

Drive content from data, not hard-coded values:

- **Config / static content** — `config.json` (above).
- **External APIs** — fetch client-side from the webapp. Keyless, CORS-enabled APIs work best
  (e.g. Open-Meteo for weather); use `getDevice().location` (lat/long) for geo-aware content.
- **Revel Digital data tables** — ⚠️ the client-sdk **`createDataTable()` shim is gadget-only**. The
  player does not inject the data-table library into a full-screen webapp, so calling it there
  throws. In a webapp, source data-table content one of two ways:
  - **Bake at deploy time (recommended)** — read the table via the Revel MCP `list_datatable_rows`
    (or `GET /datatables/{id}/rows`) and write the rows into a bundled JSON (e.g. `tenants.json`) the
    app fetches at runtime. No secret in the artifact; refresh by re-baking + redeploying. (MCP row
    *writes* need each row shaped `{ "data": { …values… } }`.)
  - **Runtime REST fetch** — `GET /datatables/{id}/rows?api_key=…` from the app for live data, but
    this embeds a scoped read key in the client artifact — only acceptable on trusted players.
  - For **real-time** data-table binding (`createDataTable().getRows()` + `on('rowUpdated', …)`),
    build a **gadget** (revel-gadget skill) instead of a webapp.

### 6. Deploy (per the user's chosen method)

See `references/deploy.md` for all three options. **Default to the MCP method** — it's the lowest
overhead (no CI, no secret, no files added) and works in any MCP-capable, skill-capable host.

- **Via the Revel Digital MCP server (recommended)** — deploy without CI, directly in the
  conversation; OAuth (browser login), so no API key is needed. Connect first (Claude Code:
  `claude mcp add --transport http reveldigital https://mcp.reveldigital.io/mcp`; other hosts: add
  the same endpoint in their MCP config). Detailed in `references/deploy.md` (Option A):
  - **A1, direct-to-CMS via presigned upload (preferred)** — `npm run build` → zip the build folder
    *contents* into `<name>.webapp` → `create_media_upload({ name, content_type, group_id? })` (MCP,
    OAuth) → `curl -T <name>.webapp -H "Content-Type: …" "<upload_url>"` (local, straight to S3, no
    auth) → `finalize_media_upload({ id })` (MCP, OAuth). The binary never passes through the MCP
    server or the model. Requires the server's upload proxy tools; if absent, the same three
    PublicAPI endpoints (`POST /media/uploads`, `PUT <upload_url>`, `POST /media/uploads/{id}/finalize`)
    can be called directly with an `X-RevelDigital-ApiKey`. Confirm the finalized record is
    webapp-typed: `mime_type` = `application/x-reveldigital-webapp`.
  - **A2, URL import (fallback)** — same build/zip, then publish to a public URL (GitHub release
    asset via `gh`) and call `import_media({ url, group_id? })`. Use when presigned upload isn't
    available.

- **GitHub Action (CI)** — only when the user wants automated redeploy on push. Create
  `.github/workflows/deploy.yml` that checks out, sets up Node, `npm ci`, `npm run build`, then calls
  `RevelDigital/webapp-action@v1` with `api-key: ${{ secrets.Revel_API_Key }}`. Include the
  **advisory** accessibility/performance report step (`continue-on-error: true`, writes to
  `$GITHUB_STEP_SUMMARY`) — it never fails the build. Tell the user to add the `Revel_API_Key`
  repository secret.

- **Direct REST upload** — a one-off local `curl` multipart upload (binary, API key, no MCP, no CI).
  See `references/deploy.md` (Option C).

### 7. Tell the user how to run and deploy

```bash
npm install
npm run dev      # develop in the browser (SDK calls fall back gracefully off-device)
npm run build    # production build to ./dist
```

To deploy: push to GitHub with the `Revel_API_Key` secret set, or run the build and upload `dist/`
via `RevelDigital/webapp-action`. The webapp then appears in the Revel Digital CMS media library and
can be scheduled to players.

## SDK API Quick Reference (React, Vue, Vanilla JS)

Applies to `@reveldigital/client-sdk`. **Angular uses `@reveldigital/player-client`** — same methods
but RxJS observables for lifecycle events; see `references/angular.md`.

```typescript
import { createPlayerClient, EventType } from "@reveldigital/client-sdk";

const client = createPlayerClient();

// Events — handle player lifecycle and commands
client.on(EventType.START, () => { /* player started */ });
client.on(EventType.STOP, () => { /* player stopped */ });
client.on(EventType.COMMAND, (data) => { /* command received: data.name, data.arg */ });
client.off(EventType.START); // remove listener (clean up on unmount)

// Device info (all Promise-based)
await client.getDeviceTime();           // device-local time, ISO8601 — use for the clock
await client.getDeviceTimeZoneName();   // e.g. "America/Chicago"
await client.getDeviceTimeZoneID();
await client.getDeviceTimeZoneOffset(); // minutes from GMT
await client.getLanguageCode();         // e.g. "en" — set on <html lang>
await client.getDeviceKey();            // unique device identifier
await client.getDevice();               // full IDevice: name, tags[], timeZone, location {…}
await client.isPreviewMode();           // true in CMS preview — fall back gracefully

// Display / content info
await client.getWidth();                // screen/zone width in pixels
await client.getHeight();               // screen/zone height in pixels
await client.getDuration();             // scheduled duration in ms
await client.getRevelRoot();            // Revel Digital root URL
await client.getCommandMap();
await client.getSdkVersion();

// Communication
client.sendCommand(name, arg);                    // send a command to the player
client.sendRemoteCommand(deviceKeys, name, arg);  // send to other devices on the account
client.callback(...args);

// Analytics
client.track(eventName, properties);
client.timeEvent(eventName);
```

### IDevice / location

`getDevice()` resolves to an `IDevice` with `name`, `registrationKey`, `deviceType`, `tags: string[]`,
`langCode`, `timeZone`, and an optional `location` →
`{ latitude, longitude, city, state, address, country, postalCode }`. Use `location` for
geo/timezone-aware content (local weather, store address, regional messaging).

## Important notes

- The SDK works both **inside the player and standalone** in a browser. Off-device, methods may
  return `null` or throw — always wrap async SDK calls in `.catch()` and design sensible fallbacks
  so `npm run dev` looks right.
- Always **clean up event listeners** (`client.off(...)` / unsubscribe) on unmount to avoid leaks.
- The build output is a plain static `dist/` — no XML, no gadget descriptor. Keep asset paths
  relative (`base: './'` in Vite) so the webapp loads regardless of where the player serves it from.
- The webapp `name` and `version` come from `package.json`; the deploy action reads them
  automatically.
- TypeScript types: `import type { PlayerClient, IDevice, IEventProperties } from '@reveldigital/client-sdk';`
  (Angular: import from `@reveldigital/player-client`).
