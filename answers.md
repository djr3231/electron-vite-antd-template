
**1. Electron 39 supports `utilityProcess`: YES**  
The `utilityProcess` API remains fully supported in Electron 39 (released October 2025, with ongoing patches like 39.8.7 as of April 2026) and the current latest versions. It was introduced in Electron 22 and continues to be actively documented without deprecation.

**Authoritative source**:  
- Official docs: https://www.electronjs.org/docs/latest/api/utility-process (includes `utilityProcess.fork`, `MessageChannelMain`/`MessagePortMain` transfer, `process.parentPort` in the child, etc.)

**Caveats**:  
- Must call `fork` only after `app.ready`.  
- Limited `stdio` options (`stdin` must be `'ignore'`; other values error).  
- `error` event is experimental; `pid` becomes `undefined` after exit.  
No version-specific breakage in v39.

**2. Comlink works with Electron's `utilityProcess` via `MessageChannelMain`: PARTIAL**  
Comlink itself has no built-in adapter for Electron's `MessagePortMain` / `MessageChannelMain` or `utilityProcess` (its core `MessagePort` support assumes the web-standard `MessagePort` with `addEventListener`/`postMessage` semantics). However, community adapters exist that bridge `MessagePortMain` (via `MessageChannelMain`) and can be adapted for `utilityProcess` scenarios. No recent dedicated tutorials or examples were found that combine Comlink + `utilityProcess` specifically, but the underlying messaging primitives are compatible.

**Key findings**:  
- **Comlink's MessagePort adapter status**: Works for standard web `MessagePort`/`MessageChannel`; Electron requires a bridge because `MessagePortMain` uses Node-style `.on('message')` / `.postMessage` (not the DOM event model).  
- **Adapters**: `comlink-adapters` provides `electronMainEndpoint` (explicitly supports `MessageChannelMain` via `messageChannelConstructor: MessageChannelMain`). It is primarily shown for main ↔ renderer but the same pattern applies to `utilityProcess` (fork + transfer `port1` to child via `child.postMessage`). Other older adapters (e.g., `comlink-electron-endpoint`, `kgullion/comlink-electron-endpoint`) also target `MessagePortMain`.  
- **GitHub / discussions**: No open blocking issues specifically for Comlink + `utilityProcess`. Older Electron issues note `MessagePortMain` non-compliance with the HTML spec (event API differences), and one 2023 bug report about bidirectional piping was filed but is not Comlink-specific.

**Caveats / known issues**:  
- Adapters internally bridge `MessagePort` ↔ `MessagePortMain`, which can have efficiency overhead (noted in `comlink-adapters`).  
- Utility-process side uses `process.parentPort` (a `MessagePortMain`), so you must wrap both ends appropriately.  
- Very few public examples exist for this exact combination (most Comlink-Electron usage is IPC/main-renderer).

**Alternative if you want zero friction**: Use raw `MessageChannelMain` + `utilityProcess.fork` + `process.parentPort` (simple and officially supported), or switch to a purpose-built RPC library like `@kunkun/kkrpc` which ships explicit `ElectronUtilityProcessIO` adapters for Main ↔ Utility Process.

**3. electron-vite (current version in the scaffold) supports bundling utility-process workers: YES**  
electron-vite has built-in, documented support for `utilityProcess` (and Node `child_process`) workers. The worker file is treated as a separate entry point via the `?modulePath` query suffix—no extra config required.

**Authoritative source & pattern**:  
- Official guide: https://electron-vite.org/guide/dev#utility-process-and-child-process (updated Dec 2025)

**Example (directly from docs)**:  
```js
// main.ts
import { utilityProcess, MessageChannelMain } from 'electron'
import forkPath from './fork?modulePath'   // ← separate bundle

const { port1, port2 } = new MessageChannelMain()
const child = utilityProcess.fork(forkPath)
child.postMessage({ message: 'hello' }, [port1])

// ... handle port2
```

**Utility-process side** (`fork.ts`):  
```js
process.parentPort.on('message', (e) => {
  const [port] = e.ports
  // ...
})
```

**Caveats**: None documented. The `?modulePath` suffix is the recommended/standard way and works out-of-the-box with Vite's bundling. The scaffold's electron-vite version supports this (docs reflect the current package).

**Summary Table**  
| Check | Result | Key Link | Notes / Alternative |
|-------|--------|----------|---------------------|
| 1. Electron 39 `utilityProcess` | YES | https://www.electronjs.org/docs/latest/api/utility-process | Fully supported; use after `app.ready` |
| 2. Comlink + `utilityProcess` / `MessageChannelMain` | PARTIAL | https://github.com/kinglisky/comlink-adapters | Adapters exist but no official/dedicated examples; use raw channels or `@kunkun/kkrpc` |
| 3. electron-vite worker bundling | YES | https://electron-vite.org/guide/dev#utility-process-and-child-process | `?modulePath` suffix = separate entry point (no config needed) |

All checks align with current (April 2026) documentation and releases. Ready for implementation once you confirm the approach for #2.
