We're adding a new capability to the template: creating large ZIP archives 
from directory contents (often from UNC network shares). This is our first 
real use case for heavy background tasks, which means we need to build the 
workers infrastructure that ARCHITECTURE.md currently marks as deferred 
("Heavy Tasks (Deferred)" section).

Do NOT start implementing yet. I need a plan first.

## Context you need to re-read before planning
- ARCHITECTURE.md — particularly the "Heavy Tasks (Deferred)" section and 
  the folder structure for src/main/
- CLAUDE.md — work style rules, package version verification, visual 
  verification rules
- IMPLEMENTATION_PLAN.md — so you understand where we are in the main 
  template build (we're mid-Step 2 or whatever step we're at when you 
  read this)

## Architectural decisions (already locked — do not reopen)

1. **Use Electron's `utilityProcess` API**, NOT `worker_threads`, NOT 
   hidden BrowserWindows. This is the officially recommended modern 
   pattern for Electron.

2. **Communication via `MessageChannelMain` + `MessagePortMain`**. The 
   utility process uses `process.parentPort`. Transfer port1 to the 
   child via `child.postMessage(msg, [port1])`.

3. **Custom type-safe RPC layer — ~100 lines, zero external dependencies**. 
   Do NOT use Comlink. Do NOT use @kunkun/kkrpc. Do NOT add any RPC 
   library. We want full control, zero risk of abandoned dependencies, 
   and a simple request/response pattern (no streams, no transferables 
   beyond the port itself).

4. **Worker files bundled via electron-vite's `?modulePath` query suffix**. 
   This is documented at https://electron-vite.org/guide/dev — verify 
   the current syntax if needed. The worker file becomes a separate 
   bundle entry point.

5. **ZIP library: `archiver`**. It's streaming-based (critical for large 
   files — we can't load 10GB into memory), pure JavaScript (no native 
   bindings, no electron-builder rebuild issues), mature and actively 
   maintained. Do NOT use adm-zip (loads everything into memory). Do NOT 
   use jszip (browser-focused). Verify current stable version with 
   `npm view archiver version` before adding it.

## Folder structure to create (aligned with ARCHITECTURE.md)
src/main/
├── workers/
│   ├── README.md                    # when to add a worker, the pattern
│   ├── zip-creator.worker.ts        # the first real worker
│   └── shared/
│       ├── worker-rpc-worker.ts     # exposeWorker() helper — worker side
│       └── worker-types.ts          # shared types between worker and main
├── services/
│   ├── worker-rpc.ts                # createWorkerClient() helper — main side
│   ├── heavy-tasks.service.ts       # wrapper that exposes zip operations to handlers
│   └── (existing services)
├── ipc/
│   └── (existing handlers — zip handler will be added to files.handlers.ts
│        or a new zip.handlers.ts, your call)
Shared types:
- `src/shared/ipc-channels.ts` — add ZIP-related channels
- `src/shared/ipc-types.ts` — add ZIP request/response types

## Functional requirements for the zip feature

- **Input**: source directory path (can be a UNC path like `\\\\server\\share\\folder`), 
  output zip file path (also possibly UNC)
- **Output**: success indication with total bytes written, or a clear 
  error message
- **Progress reporting**: the worker should emit progress events (files 
  processed, bytes processed) back through the RPC layer. How this 
  surfaces to the renderer is part of the plan — propose a mechanism 
  that fits our TanStack Query pattern (might be polling, might be an 
  event stream — you decide and justify).
- **Error handling**: clear errors for common cases — source doesn't 
  exist, no permission, disk full, invalid UNC path. Must not crash 
  the worker silently.
- **Cancellation**: nice-to-have but not required in v1. Note in the 
  plan whether you think it's worth including now or deferring.
- **Compression level**: configurable, default to 6 (balanced). Expose 
  this as a parameter.

## Non-functional requirements

- The main process must never block during zip creation. The UI must 
  remain fully responsive throughout.
- Each zip operation gets its own utility process (or we reuse a pool — 
  you decide and justify). Starting simple is fine.
- TypeScript strict mode. Worker API must be type-safe end-to-end — 
  the renderer should know what it gets back without casts.
- No `any` without a comment explaining why.

## What I want from you right now

Produce a **plan document** covering:

1. **Updated architecture decision** for ARCHITECTURE.md — a new section 
   replacing "Heavy Tasks (Deferred)" with "Heavy Tasks" containing the 
   committed implementation. Do NOT edit the file yet — show me the 
   proposed text in the plan.

2. **Dependency analysis** — what packages need to be installed, verified 
   versions from `npm view`, where they go (main process only for 
   archiver, for instance).

3. **File-by-file breakdown** — for each new or modified file, a short 
   description of its purpose and key functions/types it will export. 
   Group files into logical steps (1-5 steps, each reviewable independently).

4. **The RPC protocol design** — exactly what messages travel between 
   main and worker. Message shapes, id correlation, error propagation, 
   progress events. Show this as TypeScript interfaces.

5. **Progress reporting mechanism** — how does the renderer learn about 
   progress? Justify your choice (polling IPC, event emitter through 
   IPC, MessageChannel forwarded, etc.) against our TanStack Query + 
   Zustand architecture.

6. **Integration with existing IPC layer** — how does the zip handler 
   fit into src/main/ipc/? New file or existing? Which channel names?

7. **Example usage from renderer** — pseudo-code showing how a React 
   component would trigger a zip operation and display progress. This 
   helps me verify the DX is good before we build.

8. **Open questions** — anything you're unsure about where you want my 
   input before implementing.

## Rules for the plan itself

- Do not implement any code. Do not create any files. Plan only.
- Do not modify ARCHITECTURE.md, CLAUDE.md, or IMPLEMENTATION_PLAN.md yet.
- Verify package versions with `npm view` before including them in the plan.
- If you find something that contradicts or requires changing an existing 
  architectural decision, stop and ask — don't silently diverge.
- If an assumption in this prompt is wrong (e.g., electron-vite's 
  `?modulePath` syntax has changed, archiver has a known issue with 
  Electron 39), stop and tell me.
- Present the plan. Wait for my approval. Then we'll discuss which step 
  to start with.