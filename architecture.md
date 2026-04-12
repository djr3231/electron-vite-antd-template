# Electron Desktop App Template — Architecture Blueprint

## Purpose
A GitHub template for building internal Windows desktop applications using
Electron + Vite + React + TypeScript. Used across the organization to
bootstrap new projects with consistent architecture, theming, and deployment.

## Base Scaffold

This project was scaffolded with:
npm create @quick-start/electron@latest -- --template react-ts
The following are **inherited from the scaffold** — do not override or
redefine unless you have a specific reason:

- **Core stack versions**: Electron, electron-vite, Vite, React, TypeScript,
  electron-builder, electron-updater (see `package.json` for actual versions)
- **Tooling**: ESLint 9 flat config, Prettier, @electron-toolkit/* packages
- **Source layout**: `src/main/index.ts`, `src/preload/index.ts`,
  `src/renderer/index.html`, `src/renderer/src/` (app code)
- **TypeScript config**: `tsconfig.json` → references `tsconfig.node.json`
  (main + preload) and `tsconfig.web.json` (renderer), all extending
  `@electron-toolkit/tsconfig`
- **Vite config**: `electron.vite.config.ts` with `@renderer` path alias
- **electron-builder base config**: `electron-builder.yml`
- **Dev tooling configs**: `.prettierrc.yaml`, `.prettierignore`,
  `.editorconfig`, `eslint.config.mjs`, `.vscode/`
- **Scripts**: all `package.json` scripts (dev, build, typecheck, lint, etc.)
- **Preload bridge**: `@electron-toolkit/preload` providing `window.electron`
  API, with `contextBridge` and `contextIsolation: true`
- **Sandbox**: The scaffold sets `sandbox: false`. We accept this — for
  internal network apps with authenticated users it is an acceptable
  trade-off, and fighting `@electron-toolkit/preload` on this is not worth
  the effort. Do not change it.
- **build/ directory**: Default icon and installer assets — replace icons
  per project.

### window.electron vs window.api

`window.electron` is inherited from the scaffold's `@electron-toolkit/preload`
and provides generic electron utilities. `window.api` is our organizational
namespace containing the fixed handlers this template provides: `files.*`,
`auth.*`, `app.*`. Projects using this template may extend `window.api` with
their own handlers or create additional namespaces as their architecture
requires — that decision is left to each project.

## Architectural Boundaries (CRITICAL)

This template is **frontend only**. It does NOT include:
- `oracledb` or any database driver
- MikroORM or any ORM
- Direct Oracle connections
- Business logic

All data access goes through a **separate NestJS backend** (different repo)
over HTTPS. The Electron app is strictly:
1. UI (React in renderer process)
2. File system access for UNC network shares (main process)
3. Windows user identification via `os.userInfo()` (main process)
4. Auto-updates (main process)

## Additions: Tech Stack

These are packages and libraries **we add** on top of the scaffold.
Verify latest stable versions with `npm view <pkg> version` before installing.

| Package | Purpose |
|---|---|
| Ant Design v6 | Component library with ConfigProvider theming |
| @ant-design/icons | AntD icon set |
| TanStack Query v5 | Server state — all backend + IPC calls |
| @tanstack/react-query-devtools | Query devtools (dev only) |
| Zustand v5 | Client/UI state only |
| axios | HTTP client with auth interceptors |
| electron-log | File + console logging |

## Folder Additions

The scaffold provides the base `src/` layout. Below are the directories and
files **we add** on top of it.

> **Important:** The renderer's Vite root is `src/renderer/` (contains
> `index.html`). All renderer application code lives inside
> `src/renderer/src/`. Every renderer addition below is relative to
> `src/renderer/src/`, NOT `src/renderer/`.

```
src/
├── main/
│   ├── ipc/
│   │   ├── index.ts              # registerIpcHandlers()
│   │   ├── files.handlers.ts
│   │   └── auth.handlers.ts
│   ├── services/
│   │   ├── file.service.ts
│   │   └── updater.service.ts
│   └── config/env.ts
├── preload/
│   └── api.ts                    # Exposed API surface (extends scaffold)
├── renderer/src/                 # ← all renderer additions go HERE
│   ├── api/
│   │   ├── client.ts             # axios instance
│   │   └── queries/              # TanStack Query hooks
│   ├── components/               # Shared UI
│   ├── features/                 # Feature folders
│   │   └── example-feature/
│   │       ├── components/
│   │       ├── hooks/
│   │       ├── store.ts
│   │       └── index.ts
│   ├── hooks/useElectronApi.ts
│   ├── stores/app.store.ts
│   ├── theme/
│   │   ├── tokens.ts             # EDIT PER PROJECT
│   │   ├── theme.ts
│   │   └── README.md
│   ├── styles/
│   │   ├── global.css
│   │   └── variables.module.css
│   └── types/electron-api.d.ts
└── shared/                       # NEW directory (not in scaffold)
├── ipc-channels.ts           # Channel name constants
└── ipc-types.ts              # Payload types
Additional root-level files we add:
.npmrc                            # Artifactory registry config
TEMPLATE_SETUP.md                 # Per-project checklist
.github/workflows/build-and-publish.yml
## Additions: electron-builder.yml
```

The scaffold provides a base `electron-builder.yml`. We modify/add only
these fields (do not duplicate the rest):

```yaml
# Override appId and productName per project
appId: com.yourorg.appname
productName: App Name

# NSIS: per-user install, no admin required
win:
  target:
    - target: nsis
      arch: [x64]
nsis:
  oneClick: false
  perMachine: false
  allowToChangeInstallationDirectory: true
  createDesktopShortcut: true
  createStartMenuShortcut: true

# Auto-update from Artifactory
publish:
  provider: generic
  url: https://artifactory.yourorg.internal/artifactory/electron-apps/app-name/
  channel: latest
Additions: .npmrc (Artifactory)
registry=https://artifactory.yourorg.internal/artifactory/api/npm/npm-virtual/
@yourorg:registry=https://artifactory.yourorg.internal/artifactory/api/npm/npm-local/
always-auth=true
Replace yourorg with the actual organization subdomain when using the
template.
Key Implementation Rules
IPC Security
NEVER use nodeIntegration: true
ALWAYS use contextBridge.exposeInMainWorld
contextIsolation: true (scaffold default — do not change)
All IPC channel names as constants in src/shared/ipc-channels.ts
All IPC payloads typed in src/shared/ipc-types.ts
Preload exposes our organizational handlers under window.api, fully
typed. This is in addition to window.electron inherited from the
scaffold (see "window.electron vs window.api" section).
Authentication Flow
Main process reads os.userInfo().username and process.env.USERDOMAIN
Renderer calls window.api.auth.getCurrentUser() via IPC
axios interceptor attaches X-User-Name and X-User-Domain headers
Backend validates against Active Directory (backend's responsibility)
Frontend NEVER trusts its own user claim for authorization
File System Access
All fs operations in main process only
UNC paths (\\server\share\...) work natively with user's Windows credentials
Use fs/promises (not callbacks)
For directory watching, use chokidar (add as dep if needed)
Expose via IPC, consume in renderer through TanStack Query wrappers
Theming (Per-Project Customization)
src/renderer/src/theme/tokens.ts is the ONLY file a new project edits
for branding
Exports brandTokens object (colors, radius, font family)
theme.ts composes tokens into AntD ThemeConfig
main.tsx wraps app in
<ConfigProvider theme={lightTheme} locale={heIL} direction="rtl">
Support Hebrew: use antd/locale/he_IL, font family includes "Assistant"
or similar
State Management Rules
Server data (from backend or IPC): TanStack Query only
UI state crossing components: Zustand
Single-component UI state: useState
Forms: React Hook Form (add when needed)
Wrap IPC calls in useQuery hooks just like HTTP calls
Never store server data in Zustand
Auto-Updates
Provider: generic (HTTP)
URL: Artifactory generic repository
autoDownload: true, autoInstallOnAppQuit: true
Check on startup + every hour
Log all updater events via electron-log
Packaging
Target: NSIS, x64 only
oneClick: false, perMachine: false (no admin required)
Allow install directory change
Desktop + Start Menu shortcuts
Code signing: deferred, coordinate with infra team later
Must-Have Template Features
Global React error boundary → logs to electron-log
Single instance lock (app.requestSingleInstanceLock())
Custom or removed default menu
Logs written to %APPDATA%/<app-name>/logs
Environment-specific VITE_BACKEND_URL (dev/staging/prod)
Pre-configured ESLint + Prettier (inherited from scaffold)
TEMPLATE_SETUP.md checklist for new projects:
Remove template git history: rm -rf .git && git init
Update package.json: name, productName, description, version to 0.1.0
Create .env with VITE_BACKEND_URL
Replace "yourorg" placeholders in .npmrc
Replace "yourorg" and "app-name" placeholders in electron-builder.yml
Edit tokens.ts for branding
Replace icons in build/
Update README
Out of Scope (Do NOT Add)
Any Oracle/database code
Any ORM (MikroORM, TypeORM, Prisma, etc.)
Local databases (sqlite, lowdb, etc.)
Business logic or domain models
Backend code of any kind
Specific UI features beyond example skeleton
State management libraries beyond Zustand + TanStack Query
