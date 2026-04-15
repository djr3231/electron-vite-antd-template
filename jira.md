בטח. הנה רשימת משימות ברמת epic/story, מסודרת לפי סדר הגיוני של ביצוע. כתבתי באנגלית כי זה ה-convention שלכם ל-Jira, אבל תגיד אם אתה מעדיף בעברית.

## Electron App Template — Remaining Work

**Step 2: Main Process Services (in progress)**
- Fix `env.ts` to use `import.meta.env.MAIN_VITE_*` and `import.meta.env.DEV/PROD/MODE` instead of `process.env`
- Add dev mode guard to `updater.service.ts` (`if (appEnv.isDev) return` in `initUpdater`)

**Step 3: IPC Handlers + Main Process Wiring**
- Implement IPC handlers for files/auth/app channels
- Modify `main/index.ts`: single instance lock, menu removal, electron-log setup, `registerIpcHandlers` call, `initUpdater` call

**Workers Infrastructure + ZIP Feature**
- Review and approve Claude Code's plan (RPC protocol, progress reporting mechanism)
- Implement custom RPC layer (`worker-rpc.ts` + `worker-rpc-worker.ts`)
- Implement `zip-creator.worker.ts` using `archiver`
- Implement `heavy-tasks.service.ts` wrapper
- Wire up progress reporting through IPC event channel
- Update `ARCHITECTURE.md` "Heavy Tasks" section

**Renderer Foundation**
- Set up AntD v6 with `ConfigProvider`, Hebrew locale, RTL
- Set up theme tokens (`tokens.ts` + `theme.ts`)
- Set up TanStack Query v5 with QueryClient provider
- Set up Zustand stores skeleton
- Set up axios client with auth interceptor (`X-User-Name` / `X-User-Domain` from IPC)
- Create typed wrappers around `window.api` for renderer consumption

**Auto-Updater Configuration**
- Configure electron-updater for generic HTTP provider pointing to Artifactory
- Configure update check on startup + hourly interval
- Document Artifactory URL configuration per project

**Packaging**
- Strip mac/linux/dmg/appImage from electron-builder config
- Configure NSIS per-user install (no admin), x64 Windows only
- Verify build output

**Template Finalization**
- Write template README (setup, customization points, theming)
- Document the per-project edit points (`tokens.ts`, branding, IPC additions)
- Add example feature end-to-end (file operation through IPC + TanStack Query) as reference for new projects
- Smoke test: clone template, rename, build, run

---

## `@yourorg/oracle-client` — New Repo

**Initial Package Setup**
- Create new repo with TypeScript + NestJS structure
- Configure `package.json` with `oracledb` and `@nestjs/common` as peerDependencies
- Configure publish target to internal Artifactory

**Core Implementation**
- Port `oracle.service.ts` and `oracle.module.ts` from existing NestJS project
- Convert module to `DynamicModule` with `forRoot` and `forRootAsync`
- Replace `ConfigService` injection with `@Inject('ORACLE_CONFIG')`
- Add `withConnection` and `withTransaction` methods
- Expose `getConnection` and `getPool` as escape hatches
- Define and export public types (`OracleClientConfig`, etc.)

**Documentation**
- Write README with usage examples for all three API levels (`execute`, `withConnection`/`withTransaction`, `getConnection`/`getPool`)
- Document `connectString` formats (TNS alias, Easy Connect, full descriptor) and `TNS_ADMIN` requirements
- Document `forRoot` vs `forRootAsync` use cases

**Validation**
- Publish version 0.1.0 to Artifactory
- Migrate one existing NestJS project to use the package, remove its local `oracle/` folder, verify everything works
- Bump to 1.0.0 after successful migration

---

## NestJS Backend Template — New Repo (later)

- Scaffold NestJS template repo
- Integrate `@yourorg/oracle-client` as default DB layer
- Set up AD-based auth that validates `X-User-Name` / `X-User-Domain` from Electron clients
- Establish project structure conventions (modules, controllers, services, queries files)
- Write template README

---

## Deferred / Coordinated Separately

- CI/CD workflow (`.github/workflows/build-and-publish.yml`) for Electron template
- Code signing (coordinated with infra team)
- Splash screen (per-project, not template)
