# Revel Digital Webapp Skill

A [Claude](https://claude.com) skill that scaffolds and deploys **full-screen Revel Digital
webapps** — self-contained HTML5 apps that run on [Revel Digital](https://www.reveldigital.com)
players and talk to the player through the [`@reveldigital/client-sdk`](https://www.npmjs.com/package/@reveldigital/client-sdk)
shim. Signage best practices — a theme system, Section 508 / WCAG 2.1 AA accessibility, and
distance-readable typography — are baked into every project it generates.

> **Webapp vs. gadget.** A *webapp* is a full-screen app deployed to the Revel Digital CMS as a
> `.webapp` media asset (via [`RevelDigital/webapp-action`](https://github.com/RevelDigital/webapp-action)).
> A *gadget* is an embeddable widget defined by `gadget.yaml`, hosted on GitHub Pages — use the
> [revel-gadget skill](https://github.com/RevelDigital/reveldigital-gadget-skill) for those.
> This skill builds **webapps**: no `gadget.yaml`, no gadgetizer, no GitHub Pages.

## Supported frameworks

| Framework  | Tooling                       | SDK library                     |
|------------|-------------------------------|---------------------------------|
| **React**  | Vite + React + TypeScript     | `@reveldigital/client-sdk`      |
| Vue        | Vite + Vue 3                  | `@reveldigital/client-sdk`      |
| Angular    | Angular CLI                   | `@reveldigital/player-client`   |
| Vanilla JS | Vite + TypeScript             | `@reveldigital/client-sdk`      |

React is the recommended default.

## What gets scaffolded

- A complete, runnable project for your chosen framework.
- **Client SDK wired in** — live device clock (`getDeviceTime`), device/location details
  (`getDevice`), command handling (`EventType.COMMAND`), and graceful `isPreviewMode()` fallbacks
  so it runs both on-device and in the browser during development.
- A **`config.json`** content file + loader (webapps have no `gadget.yaml` preferences UI, so
  runtime content is driven by config).
- A **theme system** — CSS custom properties for color/spacing/typography, light + dark via
  `prefers-color-scheme`, and a one-place brand color.
- **Accessibility baseline** — semantic landmarks, `<html lang>` from the device language,
  visible focus, `prefers-reduced-motion` guards, and contrast-checked default tokens.
- **Readability for signage** — fluid `clamp()` type scale sized for viewing distance and
  overscan-safe margins so content is never clipped on a TV.
- An optional **deploy workflow** (`.github/workflows/deploy.yml`) that builds and uploads to the
  Revel Digital CMS via `RevelDigital/webapp-action`, plus an **advisory** accessibility/performance
  report that never fails the build.

## Deploying

Three paths (see the skill's `references/deploy.md`):

1. **GitHub Action (CI)** — deploy on every push via `RevelDigital/webapp-action` (uses a
   `Revel_API_Key` repo secret). Default.
2. **From Claude Code via the Revel Digital MCP server** — build and deploy without leaving the
   conversation, using OAuth (browser login) — no API key in the repo. Connect with
   `claude mcp add --transport http reveldigital https://mcp.reveldigital.io/mcp`
   ([MCP docs](https://reveldigital.github.io/reveldigital-mcp-cloudflare/)). Two approaches:
   **direct-to-CMS via presigned upload** (preferred — build → package → `create_media_upload` →
   `curl -T` the bytes straight to S3 → `finalize_media_upload`; the binary never passes through the
   MCP server or the model), or **URL import** (fallback — publish the `.webapp` as a GitHub release
   asset, then `import_media({ url })`).
3. **Direct REST upload** — a one-off local `curl` multipart upload (binary, API key, no CI).

## Installation

### Claude.ai (Web / Desktop / Mobile)

1. Download `revel-webapp-skill.zip` from the [Releases](https://github.com/RevelDigital/reveldigital-webapp-skill/releases) page.
2. Go to **Settings → Capabilities → Skills → Upload skill** and select the ZIP.
3. Ensure the **Code execution** capability is enabled.

### Claude Code (CLI) — marketplace

```bash
/plugin marketplace add RevelDigital/reveldigital-webapp-skill
/plugin install revel-webapp@reveldigital
```

Update later with `/plugin marketplace update reveldigital`.

### Claude Code (CLI) — manual

Copy the `revel-webapp-skill` folder into `~/.claude/skills/` (personal) or `.claude/skills/`
(project-level).

### Team / Enterprise

Upload `revel-webapp-skill.zip` under **Organization Settings → Capabilities → Skills**.

## Usage

Just ask Claude:

- "Create a React Revel Digital webapp called lobby-welcome with a navy brand color and the deploy workflow."
- "Scaffold a vanilla JS Revel Digital signage webapp that shows a full-screen clock and the device location."
- "Build a Vue webapp for digital signage that rotates announcements from a config file, accessible and readable from across the room."

The skill asks for your framework, app name, and brand/theme basics, then generates the full
project.

## Repo contents

```
reveldigital-webapp-skill/
├── .claude-plugin/marketplace.json   # plugin manifest
├── .github/workflows/build-skill.yml # packages + releases the skill ZIP
├── revel-webapp-skill/
│   ├── SKILL.md                      # skill instructions
│   └── references/
│       ├── react.md                  # Vite + React + TS (default)
│       ├── vue.md                    # Vite + Vue 3
│       ├── angular.md                # Angular CLI + player-client
│       ├── vanilla.md                # Vite + TS, no framework
│       ├── signage.md                # shared: theme / a11y / readability
│       └── deploy.md                 # shared: CMS deploy + advisory quality report
├── build.sh                          # build the skill ZIP locally
├── LICENSE
└── README.md
```

## Related resources

- **Client SDK docs** — https://reveldigital.github.io/reveldigital-client-sdk/
- **Deploy action** — https://github.com/RevelDigital/webapp-action
- **NPM** — [`@reveldigital/client-sdk`](https://www.npmjs.com/package/@reveldigital/client-sdk),
  [`@reveldigital/player-client`](https://www.npmjs.com/package/@reveldigital/player-client) (Angular)
- **Developer portal** — https://www.reveldigital.com

## Building the skill ZIP

```bash
./build.sh
```

## License

[MIT](LICENSE) © Revel Digital
