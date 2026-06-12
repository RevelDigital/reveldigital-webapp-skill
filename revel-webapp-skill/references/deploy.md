# Deploy Reference (Revel Digital CMS + advisory quality report)

A Revel Digital **webapp** is deployed by building it to a static `dist/` folder, zipping that into
`<name>.webapp`, and adding it to the Revel Digital CMS media library. Webapps do **not** use
gadgetizer or GitHub Pages — the only deploy target is the CMS.

## Deployment options

Pick whichever fits the user's workflow (you can offer more than one):

| Option | When to use | Auth | Overhead |
|--------|-------------|------|----------|
| **A — Revel Digital MCP server** *(recommended)* | Deploy *right now* from your agent, no CI | OAuth (browser login) | **Lowest** — no repo, secret, or workflow |
| **B — GitHub Action (CI)** | Hands-off redeploy on every `git push` | `Revel_API_Key` secret | A repo + secret + `deploy.yml` |
| **C — Direct REST upload** | One-off local deploy without MCP or CI | `Revel_API_Key` | A stored API key |

**Prefer Option A (MCP).** It has the least overhead for the creator — no GitHub repo, no stored
secret, and no `deploy.yml` to maintain — and it works from **any host that supports skills + MCP**
(Claude Code and other skill-capable agent frameworks), not just one tool. The build is packaged and
the bytes go straight to S3 via a presigned URL; only the small control-plane calls go through the
MCP server, OAuth'd via a browser login.

Use **Option B** when the user specifically wants automated redeploy wired into their git workflow.
**Option C** is the no-MCP/no-CI local fallback. Only scaffold a `deploy.yml` (Option B) when the
user opts into CI — the recommended MCP path adds no files to the project.

---

## Option A — Revel Digital MCP server (recommended)

Build and deploy a webapp **without leaving your agent** — no GitHub Actions workflow and no API key
in the repo. Works in Claude Code and any other MCP-capable, skill-capable host. The Revel Digital
MCP server authenticates via OAuth (a browser login the first time).

> **Connect the MCP server first.** Streamable HTTP endpoint: `https://mcp.reveldigital.io/mcp`
> (SSE legacy: `https://mcp.reveldigital.io/sse`). In Claude Code:
> `claude mcp add --transport http reveldigital https://mcp.reveldigital.io/mcp`, then run any tool
> once to trigger the OAuth login. Other hosts: add the same endpoint via their MCP configuration.
> Docs: https://reveldigital.github.io/reveldigital-mcp-cloudflare/

Build and package the artifact the same way for every approach below:

```bash
npm run build                       # produces dist/ (Angular: dist/<name>/)
( cd dist && zip -r "../<name>.webapp" . )   # zip CONTENTS so index.html is at the zip root
```
```powershell
# Windows PowerShell — Compress-Archive needs a .zip name, then rename
Compress-Archive -Path dist\* -DestinationPath "<name>.zip" -Force
Rename-Item "<name>.zip" "<name>.webapp"
```

### A1 — Direct-to-CMS via presigned upload (preferred)

The cleanest path: no public hosting, and the binary goes **straight to S3** — it never passes
through the MCP server or the model's token stream. It uses the PublicAPI presigned-upload endpoints:

| Step | Endpoint | Auth | Returns |
|------|----------|------|---------|
| Create slot | `POST /media/uploads` | Bearer (OAuth) **or** `X-RevelDigital-ApiKey` | `{ id, upload_url, method, headers, key, expires_at }` |
| Upload bytes | `PUT <upload_url>` | **none** (presigned URL is self-authorizing) | — |
| (Status) | `GET /media/uploads/{id}` | Bearer/apikey | `{ id, status, key, media_id, expires_at }` |
| Finalize | `POST /media/uploads/{id}/finalize` | Bearer/apikey | the full `Media` record |

Only the bytes step is unauthenticated and local; the three control-plane calls need an account
credential. Choose the auth path:

**A1a — via the Revel Digital MCP server (OAuth, preferred).** An agent **cannot** read the MCP
server's OAuth token, so it cannot call the authenticated endpoints itself — let the MCP server make
them. Requires the server to expose the upload proxy tools (`create_media_upload`,
`finalize_media_upload`, optional `get_media_upload`). If they are **not** in the connected server's
tool list, use **A1b** or **A2**.

1. **Create the slot** (MCP, OAuth):
   ```json
   create_media_upload({ "name": "<name>.webapp", "content_type": "application/zip", "group_id": "<optional>" })
   // → { "id": "...", "upload_url": "https://s3...signed...", "method": "PUT", "headers": { "Content-Type": "application/zip" }, "expires_at": "..." }
   ```
2. **Upload the bytes** (local, direct to S3 — send the `method` and exact `headers` returned above):
   ```bash
   curl -T "<name>.webapp" -H "Content-Type: application/zip" "<upload_url>"
   ```
3. **Finalize** (MCP, OAuth) → returns the `Media` record:
   ```json
   finalize_media_upload({ "id": "<id from step 1>" })
   ```
4. The webapp is now in the CMS and can be scheduled. (Poll `get_media_upload({ id })` between 2 and
   3 if needed.) **Confirm success** by checking the returned record is webapp-typed —
   `"mime_type": "application/x-reveldigital-webapp"` (a generic `application/zip` would mean the
   `.webapp` extension wasn't honored).

**A1b — direct REST with an API key (no MCP).** Same three endpoints, called directly with
`X-RevelDigital-ApiKey` (no OAuth, no MCP server). Use when the MCP upload tools aren't available and
a stored key is acceptable.

```bash
# 1. Create slot
RESP=$(curl -s -X POST "https://api.reveldigital.com/media/uploads" \
  -H "X-RevelDigital-ApiKey: $REVEL_API_KEY" -H "Content-Type: application/json" \
  -d '{"name":"<name>.webapp","content_type":"application/zip"}')
UPLOAD_URL=$(echo "$RESP" | node -e "process.stdin.once('data',d=>console.log(JSON.parse(d).upload_url))")
ID=$(echo "$RESP" | node -e "process.stdin.once('data',d=>console.log(JSON.parse(d).id))")
# 2. Upload bytes straight to S3 (no auth)
curl -T "<name>.webapp" -H "Content-Type: application/zip" "$UPLOAD_URL"
# 3. Finalize → Media record
curl -s -X POST "https://api.reveldigital.com/media/uploads/$ID/finalize" \
  -H "X-RevelDigital-ApiKey: $REVEL_API_KEY"
```

Get the API key from the Revel Digital CMS (Account → API); pass it via an env var, never commit it.

### A2 — URL import via `import_media` (fallback)

`import_media({ url, group_id? })` **fetches the file from a public URL** — it does not accept a
local binary. So publish the `.webapp` to a public URL first; a **GitHub release asset** (via `gh`)
is the most ergonomic. Use this only when presigned upload (A1) isn't available.

1. **Publish to a public URL** — create a GitHub release and read back the asset URL:

   ```bash
   gh release create webapp-v<version> "<name>.webapp" \
     --title "<name> v<version>" --notes "Webapp build for Revel Digital"
   gh release view webapp-v<version> --json assets --jq '.assets[0].url'
   ```
   (Any public host works; the URL should end in `.webapp` so the CMS infers the media type.)
2. **Choose / create a media group** (optional) — `list_media_groups` for an existing `group_id`, or
   `create_media_group`. Omit `group_id` for the default group.
3. **Import** (MCP):

   ```json
   import_media({ "url": "<public .webapp asset URL>", "group_id": "<optional group id>" })
   ```
4. **Verify** — `get_media({ id })` / `list_media`.

### Notes

- **Prefer A1a** (presigned upload via the MCP server) — OAuth, no public hosting, no stored key,
  and the bytes go straight to S3. Fall back to **A1b** (presigned + API key) when the MCP upload
  tools aren't available, or **A2** (URL import) when presigned isn't an option at all.
- Why the control-plane calls go through the MCP server in A1a: an agent cannot read the MCP
  server's OAuth token, so it can't call the authenticated `/media/uploads` endpoints directly —
  the server makes those calls. The unauthenticated `PUT` to the presigned URL is the one step the
  agent does locally.
- After the media exists, you can drive the rest of signage setup over MCP (create a
  playlist/schedule, assign to a device group) with the other Revel Digital MCP tools.

---

## Option B — GitHub Action (CI)

Choose this when the user wants automated redeploy on every push. The skill scaffolds
`.github/workflows/deploy.yml`; [`RevelDigital/webapp-action`](https://github.com/RevelDigital/webapp-action)
does the zip + upload, and an **advisory** quality report audits accessibility/performance without
ever failing the build.

### The `webapp-action` contract

| Input                   | Required | Notes                                                                 |
|-------------------------|----------|-----------------------------------------------------------------------|
| `api-key`               | **yes**  | Revel Digital API key. Store as the `Revel_API_Key` repo secret.      |
| `name`                  | no       | Webapp name. Defaults to `name` in `package.json`.                    |
| `version`               | no       | Defaults to `version` in `package.json`.                              |
| `distribution-location` | no       | Folder to package. Defaults to `dist/<name>` then `dist`.             |
| `environment`           | no       | `Production` (default) or `Development`.                              |
| `group-name`            | no       | Media group name as a regex pattern.                                  |
| `tags`                  | no       | Comma-delimited tags for smart scheduling.                            |

Output: `published` — JSON `{ version, media }` describing the uploaded media file.

**Match the build output to `distribution-location`.** Vite (React/Vue/Vanilla) builds to `dist/`,
so the default works. Angular builds to `dist/<name>/<browser?>` — set `distribution-location`
explicitly to the folder that contains `index.html` (see `angular.md`).

### `.github/workflows/deploy.yml`

```yaml
name: Deploy Webapp to Revel Digital

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22.x'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build webapp
        run: npm run build

      # --- Advisory quality report: NEVER fails the build (continue-on-error) ---
      - name: Accessibility & performance report (advisory)
        id: quality
        continue-on-error: true
        run: |
          npm i -g pa11y-ci serve
          serve -s dist -l 4173 &
          SERVER_PID=$!
          sleep 3
          echo "## Quality report (advisory)" >> "$GITHUB_STEP_SUMMARY"
          echo "" >> "$GITHUB_STEP_SUMMARY"
          echo '### Accessibility (pa11y / WCAG2AA)' >> "$GITHUB_STEP_SUMMARY"
          echo '```' >> "$GITHUB_STEP_SUMMARY"
          pa11y-ci --json "http://localhost:4173" \
            > pa11y.json 2>&1 || true
          node -e "const r=require('./pa11y.json');const t=r.total??0,e=r.errors??0;console.log(t-e+' passed, '+e+' issues across '+ (r.results?Object.keys(r.results).length:0) +' page(s).');if(e){for(const u in r.results){for(const i of r.results[u]){console.log('- ['+i.type+'] '+i.message+(i.selector?' @ '+i.selector:''));}}}" \
            >> "$GITHUB_STEP_SUMMARY" 2>&1 || echo "pa11y produced no parseable output (advisory)." >> "$GITHUB_STEP_SUMMARY"
          echo '```' >> "$GITHUB_STEP_SUMMARY"
          echo "" >> "$GITHUB_STEP_SUMMARY"
          echo "_Advisory only — these findings do not block deployment._" >> "$GITHUB_STEP_SUMMARY"
          kill $SERVER_PID || true

      - name: Deploy to Revel Digital CMS
        id: publish
        uses: RevelDigital/webapp-action@v1
        with:
          api-key: ${{ secrets.Revel_API_Key }}
          environment: ${{ github.head_ref || github.ref_name }}
          # name / version are read from package.json
          # distribution-location: dist   # set explicitly for Angular (see angular.md)

      - name: Summarize publish
        if: steps.publish.outcome == 'success'
        run: |
          echo "## Published" >> "$GITHUB_STEP_SUMMARY"
          echo "Webapp uploaded to the Revel Digital CMS." >> "$GITHUB_STEP_SUMMARY"
```

### Required secret

Add the API key as a repository secret named **`Revel_API_Key`**
(Settings → Secrets and variables → Actions → New repository secret). Get the key from the Revel
Digital CMS (Account → API). Never commit the key.

### Notes on the advisory report

- The step uses `continue-on-error: true` and ends every command with `|| true`, so a failing or
  empty audit can never block the deploy — this is intentional and matches the skill's
  **advisory-only** quality policy.
- It uses `pa11y-ci` (WCAG2AA, accessibility-focused) because accessibility is the priority for
  signage. To also capture performance/SEO/best-practices, swap or add a Lighthouse step, e.g.
  `treosh/lighthouse-ci-action@v12` pointed at `http://localhost:4173`, likewise with
  `continue-on-error: true`. Keep all quality steps advisory.
- Results are written to the GitHub **job summary** (`$GITHUB_STEP_SUMMARY`) so they're visible on
  the run page without failing anything.
- Teams that later want to *enforce* a gate can remove `continue-on-error` from a specific check —
  but the default scaffold never does.

---

## Option C — Direct REST upload (local binary, no MCP, no CI)

The same multipart upload the GitHub Action performs, run locally. Uploads the binary directly but
uses an API key rather than OAuth. Useful for a quick one-off when MCP isn't connected.

```bash
npm run build
( cd dist && zip -r "../<name>.webapp" . )

curl -X POST "https://api.reveldigital.com/media" \
  -H "X-Reveldigital-Apikey: $REVEL_API_KEY" \
  -F "file=@<name>.webapp" \
  -F "name=<name>.webapp" \
  -F "is_shared=false" \
  -F "tags=version=<version>"$'\n'"env=Production"
```

Get the API key from the Revel Digital CMS (Account → API). Pass it via an env var; never commit it.

## Local advisory quality check (any option)

```bash
npm run build
npx serve -s dist -l 4173 &
npx pa11y-ci --json http://localhost:4173   # advisory accessibility audit
```
