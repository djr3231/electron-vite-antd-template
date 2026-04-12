# Project Rules for Claude Code

## Source of Truth
ARCHITECTURE.md is the authoritative blueprint for this template.
Before making any structural decision, re-read the relevant section.
If something isn't covered there, ask — don't improvise.

## Git Workflow
- After completing any meaningful step, commit with a clear message.
- Use conventional commit prefixes: feat:, fix:, chore:, docs:, refactor:
- Never run `git reset`, `git push --force`, or rewrite history without asking.
- Auto-commit hooks are configured in .claude/settings.json — do not disable them.

## Package Versions
- Never trust version numbers from memory or from ARCHITECTURE.md.
- Always verify with `npm view <package> version` before adding to package.json.
- Use latest stable unless I explicitly say otherwise. No beta/rc/next tags.
- If a package has known incompatibility with another (e.g. AntD + React 19),
  warn me before adding it.

## Work Style
- Work in small, reviewable steps. Stop and show me after each step.
- Do not run `npm install` unless I explicitly ask.
- Do not create files outside the structure defined in ARCHITECTURE.md.
- When in doubt between two approaches, ask — don't pick silently.
- If a step produces more than ~10 files, propose splitting it first.

## Visual and Runtime Verification
- Do NOT run `npm run dev`, start the app, take screenshots, or attempt
  to visually verify UI changes. These actions consume excessive tokens
  and slow the workflow.
- For verification of changes that produce visible output, hand off to me:
  state clearly what should be checked, what the expected result is, and
  wait for my confirmation before proceeding.
- You MAY run non-interactive checks: `npm run typecheck`, `npm run lint`,
  `npm run build`. These are fast and produce text output suitable for
  your context.
- If a build/typecheck/lint passes but visual behavior is uncertain, hand
  off to me — don't try to "verify by running".

## Code Standards
- TypeScript strict mode. No `any` without a comment explaining why.
- All IPC channels as constants in src/shared/ipc-channels.ts.
- All IPC payloads typed in src/shared/ipc-types.ts.
- Never use `nodeIntegration: true` or disable `contextIsolation`.
- The scaffold sets `sandbox: false` — accept this, do not change it.
- Never import `fs`, `path`, `os`, or other Node built-ins from renderer code.
- All HTTP and IPC calls from renderer go through TanStack Query hooks.

## Out of Scope (Do NOT Add)
- oracledb or any database driver
- Any ORM (MikroORM, TypeORM, Prisma, etc.)
- Backend code of any kind
- Business logic or domain models
- Local databases (sqlite, lowdb, etc.)
- State management libraries beyond Zustand + TanStack Query
See ARCHITECTURE.md "Out of Scope" section for full rationale.

## Language
Respond to me in Hebrew when I write in Hebrew, English when I write in English.
Code, comments, commit messages, and file contents: English only.

## Component Design and Styling

### Use Ant Design Components, Not Custom Replacements
- Before creating any custom component, search the AntD docs for an 
  existing component or composition that solves the problem.
- Examples of patterns that have a built-in AntD solution and must NOT 
  be hand-built:
  - Layout structure → use Layout, Layout.Header, Layout.Sider, Layout.Content
  - Grids and columns → use Row + Col with the 24-column system
  - Forms → use Form, Form.Item, Form.List
  - Tables → use Table with built-in features (sorting, filtering, pagination)
  - Modals, Drawers, Popovers, Tooltips → use AntD components, never build from div
  - Spacing between elements → use Space and Space.Compact, not custom margins
  - Typography → use Typography.Title, Typography.Text, Typography.Paragraph
  - Loading states → use Spin and Skeleton
  - Empty states → use Empty
  - Notifications → use message and notification APIs
- Use AntD's built-in props (size, type, variant, status) before reaching 
  for custom CSS.
- If a custom component is genuinely needed, justify why no AntD 
  composition works before writing it.

### Layout Must Be Responsive and Flow-Based
- Always use flexbox (display: flex) or AntD's Layout/Row/Col for arrangement.
- Components must size relative to their parent, not absolute pixels:
  - Use percentages, fr units, flex-grow/flex-shrink/flex-basis
  - Use AntD's responsive Col props (xs, sm, md, lg, xl)
  - Avoid fixed widths and heights unless the design explicitly requires them
- Never position elements with absolute pixel coordinates (top, left in px)
  unless absolutely necessary (e.g., positioned overlays).
- Avoid hardcoded magic numbers for spacing — use the theme tokens 
  (token.padding, token.margin, token.borderRadius) or AntD's Space component.
- Components should adapt to container width changes without breaking.
- Test mental model: if the parent shrinks by 30%, does the layout 
  reflow gracefully or break? It must
