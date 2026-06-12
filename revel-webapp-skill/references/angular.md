# Angular Webapp Reference

Scaffold a full-screen Revel Digital **webapp** using Angular + the **`@reveldigital/player-client`**
library. Read `references/signage.md` and `references/deploy.md` alongside this file — `theme.css`,
the accessibility rules, and the deploy workflow come from there.

**Important:** Angular uses `@reveldigital/player-client` (an Angular-native library with an
injectable `PlayerClientService` and RxJS lifecycle observables), **not** `@reveldigital/client-sdk`.

This is a **webapp**, not a gadget: no `gadget.yaml`, no gadgetizer, no GitHub Pages. The build
output is a static folder the CMS deploy action uploads.

## Scaffolding approach

```bash
ng new {app-name} --directory ./ --standalone --style=css --routing=false
ng add @reveldigital/player-client@latest
```

The schematic adds the library and wires the providers. **You do not need its gadget/GitHub-Pages
bits for a webapp** — ignore any `gadget.yaml`/`build:gadget`/`deploy:gadget` it adds, and use the
`angular.json` below.

**Critical — fix the builder after `ng new`:** Angular 17's `ng new` uses the `application` builder,
which outputs to `dist/{app-name}/browser/`. Replace the generated `angular.json` with the one below,
which uses the `browser` builder so output lands in `dist/{app-name}/` (the folder that contains
`index.html`). This keeps the deploy `distribution-location` simple.

## Project Structure

```
{app-name}/
├── src/
│   ├── app/
│   │   ├── app.component.ts
│   │   ├── app.component.html
│   │   ├── app.component.css
│   │   ├── app.config.ts
│   │   └── config.ts            # config.json loader + types
│   ├── assets/
│   │   └── config.json          # runtime content (editable without rebuilding)
│   ├── index.html
│   ├── main.ts
│   ├── theme.css                # from references/signage.md (verbatim)
│   └── styles.css
├── .github/
│   └── workflows/
│       └── deploy.yml           # only if deploying (from references/deploy.md)
├── .browserslistrc
├── angular.json
├── package.json
└── tsconfig.json
```

> Copy `src/theme.css` **verbatim** from `references/signage.md` (section 1).

## package.json

```json
{
  "name": "{app-name}",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "ng": "ng",
    "start": "ng serve",
    "build": "ng build"
  },
  "dependencies": {
    "@angular/animations": "^17.0.0",
    "@angular/common": "^17.0.0",
    "@angular/compiler": "^17.0.0",
    "@angular/core": "^17.0.0",
    "@angular/platform-browser": "^17.0.0",
    "@angular/platform-browser-dynamic": "^17.0.0",
    "@reveldigital/player-client": "latest",
    "rxjs": "~7.8.0",
    "tslib": "^2.3.0",
    "zone.js": "~0.14.0"
  },
  "devDependencies": {
    "@angular-devkit/build-angular": "^17.0.0",
    "@angular/cli": "^17.0.0",
    "@angular/compiler-cli": "^17.0.0",
    "typescript": "~5.2.0"
  }
}
```

Differs from React/Vue/Vanilla: uses `@reveldigital/player-client` (not `@reveldigital/client-sdk`),
and there is no gadgetizer/ghpages tooling.

## angular.json

```json
{
  "$schema": "./node_modules/@angular/cli/lib/config/schema.json",
  "version": 1,
  "newProjectRoot": "projects",
  "projects": {
    "{app-name}": {
      "projectType": "application",
      "root": "",
      "sourceRoot": "src",
      "prefix": "app",
      "architect": {
        "build": {
          "builder": "@angular-devkit/build-angular:browser",
          "options": {
            "outputPath": "dist/{app-name}",
            "index": "src/index.html",
            "main": "src/main.ts",
            "tsConfig": "tsconfig.json",
            "baseHref": "./",
            "assets": [
              { "glob": "**/*", "input": "src/assets", "output": "/assets" }
            ],
            "styles": ["src/theme.css", "src/styles.css"],
            "scripts": []
          },
          "configurations": {
            "production": {
              "budgets": [
                { "type": "initial", "maximumWarning": "500kB", "maximumError": "1MB" }
              ],
              "outputHashing": "all"
            },
            "development": {
              "optimization": false,
              "extractLicenses": false,
              "sourceMap": true
            }
          },
          "defaultConfiguration": "production"
        },
        "serve": {
          "builder": "@angular-devkit/build-angular:dev-server",
          "configurations": {
            "production": { "buildTarget": "{app-name}:build:production" },
            "development": { "buildTarget": "{app-name}:build:development" }
          },
          "defaultConfiguration": "development"
        }
      }
    }
  }
}
```

**Use `@angular-devkit/build-angular:browser`** so the output is `dist/{app-name}/` (no `/browser/`
subfolder). `theme.css` and `styles.css` are both global styles.

## tsconfig.json

```json
{
  "compileOnSave": false,
  "compilerOptions": {
    "outDir": "./dist/out-tsc",
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "noImplicitOverride": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "esModuleInterop": true,
    "target": "ES2022",
    "module": "ES2022",
    "moduleResolution": "node",
    "importHelpers": true,
    "sourceMap": true,
    "declaration": false,
    "experimentalDecorators": true,
    "lib": ["ES2022", "dom"],
    "skipLibCheck": true,
    "useDefineForClassFields": false,
    "resolveJsonModule": true
  },
  "angularCompilerOptions": {
    "enableI18nLegacyMessageIdFormat": false,
    "strictInjectionParameters": true,
    "strictInputAccessModifiers": true,
    "strictTemplates": true
  }
}
```

## .browserslistrc

```
last 2 Chrome versions
last 1 Firefox version
last 2 Edge major versions
last 2 Safari major versions
last 2 iOS major versions
Firefox ESR
not dead
```

## src/index.html

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>{app-name}</title>
  <base href="./">
  <meta name="viewport" content="width=device-width, initial-scale=1">
</head>
<body>
  <app-root></app-root>
</body>
</html>
```

Omit `data-theme` to follow the device, or set `data-theme="light"`/`"dark"` on `<html>` for the
chosen appearance.

## src/main.ts

```typescript
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';
import { appConfig } from './app/app.config';

bootstrapApplication(AppComponent, appConfig)
  .catch((err) => console.error(err));
```

## src/app/app.config.ts

```typescript
import { ApplicationConfig } from '@angular/core';

export const appConfig: ApplicationConfig = {
  providers: []
};
```

## src/app/config.ts

```typescript
export interface AppConfig {
  title: string;
  messages: string[];
  rotateSeconds: number;
  showClock: boolean;
  showLocation: boolean;
}

const DEFAULT_CONFIG: AppConfig = {
  title: 'Welcome',
  messages: ['Configure src/assets/config.json to set your content.'],
  rotateSeconds: 8,
  showClock: true,
  showLocation: true,
};

export async function loadConfig(): Promise<AppConfig> {
  try {
    const res = await fetch('assets/config.json', { cache: 'no-store' });
    if (!res.ok) return DEFAULT_CONFIG;
    return { ...DEFAULT_CONFIG, ...(await res.json()) };
  } catch {
    return DEFAULT_CONFIG;
  }
}
```

## src/assets/config.json

```json
{
  "title": "Welcome",
  "messages": [
    "Edit src/assets/config.json to change this content.",
    "Designed for accessibility and distance readability."
  ],
  "rotateSeconds": 8,
  "showClock": true,
  "showLocation": true
}
```

## src/app/app.component.ts

`PlayerClientService` is injectable; lifecycle/commands use RxJS observables (`onCommand$`) rather
than `client.on()`. The clock polls `getDeviceTime()`; everything falls back gracefully off-device.

```typescript
import { Component, OnInit, OnDestroy } from '@angular/core';
import { CommonModule } from '@angular/common';
import { Subscription } from 'rxjs';
import { PlayerClientService } from '@reveldigital/player-client';
import { loadConfig, type AppConfig } from './config';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [CommonModule],
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css']
})
export class AppComponent implements OnInit, OnDestroy {
  config: AppConfig | null = null;
  msgIndex = 0;
  clock = '--:--';
  place: string | null = null;
  deviceName: string | null = null;
  isPreview = false;
  lastCommand: string | null = null;

  private subs = new Subscription();
  private clockId?: number;
  private rotateId?: number;

  constructor(public client: PlayerClientService) {
    this.subs.add(
      this.client.onCommand$.subscribe((cmd: { name?: string; arg?: string }) => {
        this.lastCommand = `${cmd?.name ?? 'command'}${cmd?.arg ? `: ${cmd.arg}` : ''}`;
      })
    );
  }

  async ngOnInit() {
    this.config = await loadConfig();

    if (this.config.messages.length > 1) {
      const ms = Math.max(2, this.config.rotateSeconds) * 1000;
      this.rotateId = window.setInterval(() => {
        this.msgIndex = (this.msgIndex + 1) % this.config!.messages.length;
      }, ms);
    }

    this.client.isPreviewMode().then((p) => (this.isPreview = Boolean(p))).catch(() => {});

    this.client.getLanguageCode()
      .then((lang) => { if (lang) document.documentElement.lang = lang; })
      .catch(() => {});

    this.client.getDevice().then((device: any) => {
      if (!device) return;
      this.deviceName = device.name ?? null;
      if (this.config?.showLocation && device.location) {
        this.place = [device.location.city, device.location.state].filter(Boolean).join(', ') || null;
      }
    }).catch(() => {});

    const tick = async () => {
      let when: Date;
      try {
        const iso = await this.client.getDeviceTime();
        when = iso ? new Date(iso) : new Date();
      } catch {
        when = new Date();
      }
      this.clock = new Intl.DateTimeFormat(undefined, {
        hour: '2-digit', minute: '2-digit', second: '2-digit',
      }).format(when);
    };
    tick();
    this.clockId = window.setInterval(tick, 1000);
  }

  ngOnDestroy() {
    this.subs.unsubscribe();
    if (this.clockId) window.clearInterval(this.clockId);
    if (this.rotateId) window.clearInterval(this.rotateId);
  }
}
```

## src/app/app.component.html

```html
<main class="signage-root" *ngIf="config; else loadingTpl">
  <header class="header">
    <h1>{{ config.title }}</h1>
    <span *ngIf="isPreview" class="badge">Preview</span>
  </header>

  <section *ngIf="config.showClock" class="clock" aria-live="off" aria-label="Current time">
    {{ clock }}
  </section>

  <section class="message accent animate-fade" aria-live="polite">
    <p>{{ config.messages[msgIndex] }}</p>
  </section>

  <footer class="footer">
    <span *ngIf="config.showLocation && place" class="footer-item">📍 {{ place }}</span>
    <span *ngIf="deviceName" class="footer-item">{{ deviceName }}</span>
    <span *ngIf="lastCommand" class="footer-item">Last command — {{ lastCommand }}</span>
  </footer>
</main>

<ng-template #loadingTpl>
  <main class="signage-root"><h1>Loading…</h1></main>
</ng-template>
```

## src/app/app.component.css

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

## src/styles.css

`theme.css` is loaded first via the `angular.json` `styles` array; keep app-wide tweaks here.

```css
html, body { height: 100%; }
```

## PlayerClientService API (Angular-specific)

Nearly identical to `client-sdk`, but events use RxJS observables:

```typescript
client.onReady$    // Subject<boolean>
client.onStart$    // Subject<void>
client.onStop$     // Subject<void>
client.onCommand$  // Subject<ICommand> — { name, arg }
client.onConfig$   // Subject — config/preview changes
```

All device/data methods (`getDeviceTime()`, `getDevice()`, `getLanguageCode()`, `isPreviewMode()`,
`sendCommand()`, `track()`, etc.) match the client-sdk.

## Build, Development & Deploy

```bash
npm install
npm start                              # dev server (SDK falls back gracefully off-device)
npm run build                          # production build to dist/{app-name}/
```

For deployment, set `distribution-location: dist/{app-name}` in the webapp-action step (see
`references/deploy.md`). Because the action's default is `dist/<package.json name>`, this also works
without setting it explicitly as long as the project name matches.
