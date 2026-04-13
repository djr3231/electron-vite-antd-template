# Implementation Plan — Electron App Template

## Context
The project was scaffolded with `npm create @quick-start/electron@latest -- --template react-ts`.
No code has been modified. ARCHITECTURE.md (now final) defines the delta we must apply.
This plan bridges the gap between scaffold state and ARCHITECTURE.md requirements.

## GAP ANALYSIS

### Already Present (keep as-is)
- Core stack versions (Electron 39, Vite 7, React 19, TS 5.9) — package.json
- electron-updater dependency — package.json
- ESLint 9 flat config — eslint.config.mjs
- Prettier config — .prettierrc.yaml, .prettierignore
- EditorConfig — .editorconfig
- TypeScript project references — tsconfig.json
- Vite config with @renderer alias — electron.vite.config.ts
- Preload bridge (window.electron) — src/preload/index.ts
- sandbox: false, contextIsolation: true — src/main/index.ts
- VSCode configs — .vscode/
- Build icons directory — build/
- npm scripts — package.json
- autoHideMenuBar: true — src/main/index.ts
- External link handler — src/main/index.ts
- HMR/production URL loading — src/main/index.ts
- src/renderer/index.html — keep structure (CSP may need update)
- src/renderer/src/env.d.ts — Vite client types

### Already Present (needs modification)

| File | Changes |
|---|---|
| src/main/index.ts | Add single instance lock, remove default menu, integrate IPC handlers + updater + electron-log, remove demo ping handler |
| src/preload/index.ts | Import typed API from new api.ts instead of {} |
| src/preload/index.d.ts | Replace api: unknown with typed ElectronApi interface |
| src/renderer/src/main.tsx | Wrap with ConfigProvider (antd, heIL, RTL), QueryClientProvider, ErrorBoundary. Change CSS import |
| src/renderer/src/App.tsx | Replace demo content with AntD Layout skeleton |
| electron-builder.yml | Add NSIS per-user settings, update publish URL, remove mac/linux/dmg/appImage sections |
| dev-app-update.yml | Update URL to Artifactory placeholder |
| tsconfig.node.json | Add src/shared/**/* to include |
| tsconfig.web.json | Add src/shared/**/* to include |
| src/renderer/index.html | Update CSP for AntD fonts, update <title> |

### Missing (needs to be added)
- Shared: src/shared/ipc-channels.ts, src/shared/ipc-types.ts
- Main: src/main/ipc/{index,files.handlers,auth.handlers}.ts, src/main/services/{file.service,updater.service}.ts, src/main/config/env.ts
- Preload: src/preload/api.ts
- Renderer: api/client.ts, api/queries/, components/ErrorBoundary.tsx, hooks/useElectronApi.ts, stores/app.store.ts, theme/{tokens,theme}.ts, theme/README.md, styles/{global,variables.module}.css, types/electron-api.d.ts, features/example-feature/
- Root: .npmrc, TEMPLATE_SETUP.md

### Present but Not in ARCHITECTURE.md

| Item | Decision |
|---|---|
| src/renderer/src/assets/electron.svg | Remove — demo asset |
| src/renderer/src/assets/wavy-lines.svg | Remove — demo asset |
| src/renderer/src/assets/base.css | Remove — replaced by styles/global.css |
| src/renderer/src/assets/main.css | Remove — replaced by styles/global.css |
| src/renderer/src/components/Versions.tsx | Remove — demo content |
| ipcMain.on('ping') in main | Remove — replaced by proper IPC handlers |
| build/entitlements.mac.plist | Remove — Windows-only focus |
| build/icon.icns | Remove — macOS icon, not needed |
| mac/linux/dmg/appImage in electron-builder.yml | Remove — Windows-only |
| CLAUDE.md, .claude/ | Keep — dev tooling, not shipped |
| ARCHITECTURE.md | Keep — reference doc, excluded from build |
| README.md | Keep — update per project |
| src/renderer/src/assets/ directory | Keep — empty after demo removal, useful for future assets |
| .github/workflows/build-and-publish.yml | Defer — CI/CD is a separate concern |

## IMPLEMENTATION PLAN

### Step 0: Package Installation
**Requires npm install — I will list exact packages. You run the install.**

- dependencies: antd, @ant-design/icons, @tanstack/react-query, zustand, axios, electron-log
- devDependencies: @tanstack/react-query-devtools
- Verify versions with `npm view <pkg> version` before installing.

### Step 1: Shared IPC Foundation + tsconfig updates
**New: 2 files | Modified: 2 files**

- Create src/shared/ipc-channels.ts — channel name constants organized by namespace (FILES, AUTH, APP)
- Create src/shared/ipc-types.ts — TypeScript interfaces for all IPC payloads (ReadDirRequest/Response, CurrentUser, AppEnv, etc.)
- Modify tsconfig.node.json — add "src/shared/**/*" to include
- Modify tsconfig.web.json — add "src/shared/**/*" to include

### Step 2: Main Process Services
**New: 3 files**

- Create src/main/config/env.ts — appEnv object reading VITE_BACKEND_URL, environment detection
- Create src/main/services/file.service.ts — fs/promises wrappers (readDir, readFile, writeFile, stat) with UNC path support
- Create src/main/services/updater.service.ts — wraps electron-updater with electron-log, autoDownload, hourly checks

### Step 3: Main Process IPC Handlers + Entry Point
**New: 3 files | Modified: 1 file**

- Create src/main/ipc/files.handlers.ts — ipcMain.handle for file channels, delegates to FileService
- Create src/main/ipc/auth.handlers.ts — ipcMain.handle for GET_CURRENT_USER using os.userInfo()
- Create src/main/ipc/index.ts — registerIpcHandlers() aggregator
- Modify src/main/index.ts:
  - Import and configure electron-log (logs to %APPDATA%/<app-name>/logs)
  - Add app.requestSingleInstanceLock() (quit if not acquired)
  - Remove default menu (Menu.setApplicationMenu(null))
  - Remove demo ipcMain.on('ping')
  - Call registerIpcHandlers() in app.whenReady()
  - Call initUpdater() after window creation

### Step 4: Preload API Surface
**New: 1 file | Modified: 2 files**

- Create src/preload/api.ts — typed API object with files.*, auth.*, app.* namespaces, each calling ipcRenderer.invoke with shared channel constants
- Modify src/preload/index.ts — import api from ./api instead of const api = {}
- Modify src/preload/index.d.ts — replace api: unknown with typed ElectronApi interface

### Step 5: Renderer Types, Hooks, and Stores
**New: 3 files**

- Create src/renderer/src/types/electron-api.d.ts — ElectronApi interface matching preload surface, global Window augmentation
- Create src/renderer/src/hooks/useElectronApi.ts — returns typed window.api
- Create src/renderer/src/stores/app.store.ts — minimal Zustand v5 store for UI state

### Step 6: Theme System and Styles
**New: 5 files | Removed: 4 files | Modified: 1 file**

- Create src/renderer/src/theme/tokens.ts — brandTokens (primaryColor, borderRadius, fontFamily with "Assistant")
- Create src/renderer/src/theme/theme.ts — composes tokens into AntD ThemeConfig, exports lightTheme
- Create src/renderer/src/theme/README.md — short theming guide
- Create src/renderer/src/styles/global.css — minimal reset, font stack, scrollbar, full-viewport #root
- Create src/renderer/src/styles/variables.module.css — spacing, z-index, transition custom properties
- Remove src/renderer/src/assets/base.css
- Remove src/renderer/src/assets/main.css
- Remove src/renderer/src/assets/electron.svg
- Remove src/renderer/src/assets/wavy-lines.svg
- Modify src/renderer/index.html — update CSP (font-src 'self' data:), update <title>

### Step 7: Renderer Core Setup (Providers, Error Boundary, Axios)
**New: 3 files | Modified: 2 files | Removed: 1 file**

- Create src/renderer/src/components/ErrorBoundary.tsx — React error boundary, logs via electron-log/renderer, renders AntD Result fallback with retry
- Create src/renderer/src/api/client.ts — axios instance with VITE_BACKEND_URL base, async auth interceptor (caches user after first IPC call), attaches X-User-Name/X-User-Domain headers
- Create src/renderer/src/api/queries/useCurrentUser.ts — example TanStack Query hook wrapping window.api.auth.getCurrentUser()
- Modify src/renderer/src/main.tsx — import global.css, wrap in ErrorBoundary > QueryClientProvider > ConfigProvider(lightTheme, heIL, "rtl") > App, add ReactQueryDevtools
- Modify src/renderer/src/App.tsx — replace demo with AntD Layout/Layout.Header/Layout.Content skeleton
- Remove src/renderer/src/components/Versions.tsx

### Step 8: Example Feature
**New: 4 files**

- Create src/renderer/src/features/example-feature/index.ts — barrel export
- Create src/renderer/src/features/example-feature/store.ts — Zustand slice for feature-local UI state
- Create src/renderer/src/features/example-feature/components/ExampleCard.tsx — AntD Card using theme, query hook, store
- Create src/renderer/src/features/example-feature/hooks/useExampleData.ts — TanStack Query hook wrapping IPC call (e.g., window.api.app.getVersion())

### Step 9: Build Config and Root Files
**Modified: 2 files | New: 2 files | Removed: 2 files**

- Modify electron-builder.yml:
  - Update appId → com.yourorg.appname
  - Add NSIS: oneClick: false, perMachine: false, allowToChangeInstallationDirectory: true, createStartMenuShortcut: true
  - Update publish.url → Artifactory placeholder, add channel: latest
  - Remove mac, dmg, linux, appImage sections
- Modify dev-app-update.yml — update URL to Artifactory placeholder
- Remove build/entitlements.mac.plist — macOS-only
- Remove build/icon.icns — macOS-only
- Create .npmrc — Artifactory registry config with yourorg placeholders
- Create TEMPLATE_SETUP.md — per-project checklist (git reset, package.json, .env, placeholders, tokens, icons, README)

### Step 10: Verification
**No file changes — readonly**

- Run `npm run typecheck` — verify no type errors
- Run `npm run lint` — verify ESLint passes
- Run `npm run build` — verify full build pipeline
- You produce a manual check list for me to verify when I run `npm run dev` myself (AntD theme applied, RTL layout, error boundary works, demo content gone, etc.). Do NOT run dev or attempt visual checks yourself.

## Dependency Graph
Step 0 (packages — user runs npm install)
│
Step 1 (shared IPC foundation)
│
├──► Step 2 (main services)
│       │
│       └──► Step 3 (IPC handlers + main entry mods)
│
└──► Step 4 (preload API)
│
└──► Step 5 (renderer types + hooks + stores)
│
├──► Step 6 (theme + styles)
│
└──► Step 7 (providers + error boundary + axios) ← depends on 5 + 6
│
└──► Step 8 (example feature)
│
└──► Step 9 (build config + root files)
│
└──► Step 10 (verification)
## Technical Notes

- **electron-log in renderer**: import from `electron-log/renderer` — auto-transports to main process via IPC
- **Axios auth interceptor**: async interceptor, cache user after first IPC call to avoid repeated roundtrips
- **Zustand v5**: use `create<StateType>()(...)` curried form
- **AntD v6 + CSP**: v6 uses CSS-in-JS; current style-src 'unsafe-inline' covers this; may need font-src 'self' data: for icon fonts — verify in Step 10
- **src/shared/ in both tsconfigs**: both main+preload and renderer import from shared
