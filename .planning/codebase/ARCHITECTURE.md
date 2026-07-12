<!-- refreshed: 2026-07-12 -->
# Architecture

**Analysis Date:** 2026-07-12

## System Overview

```text
┌─────────────────────────────────────────────────────────────────────┐
│                         React 19 Frontend (WKWebView)                │
├───────────────┬───────────────────────────┬─────────────────────────┤
│  Sidebar /     │   MainArea (tabs, PTY,    │  RightPanel (git diff, │
│  UnifiedBar    │   editor, diff, preview)  │  file tree, run output) │
│ `src/components│  `src/components/task/`  │ `src/components/task/  │
│  /sidebar`     │                           │  RightPanel.tsx`       │
└───────┬────────┴─────────────┬─────────────┴───────────┬────────────┘
        │                      │                          │
        ▼                      ▼                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                Zustand Stores (client-side state)                    │
│   `src/store/app.ts` (tasks/projects/tabs) `src/store/ui.ts`         │
│   `src/store/prefs.ts` (persisted prefs)  `src/store/scriptRuns.ts`  │
└───────────────────────────────┬───────────────────────────────────────┘
                                 │  invoke() / listen()
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    IPC Boundary — `src/lib/ipc.ts`                   │
│         typed wrappers around every #[tauri::command]                │
└───────────────────────────────┬───────────────────────────────────────┘
                                 │  Tauri invoke/event bridge
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  Rust Backend — `src-tauri/src/lib.rs`               │
│   (single monolithic module: PTY, project/task IO, settings,         │
│    scripts, git, sandbox orchestration, spotlight)                   │
│   plus focused sibling modules:                                      │
│   `sandbox.rs` `proxy.rs` `repo_config.rs` `shell_env.rs`             │
│   `automation.rs`                                                    │
└───────────────────────────────┬───────────────────────────────────────┘
                                 │
                 ┌───────────────┼────────────────┐
                 ▼               ▼                ▼
        ┌────────────┐  ┌────────────────┐ ┌─────────────────┐
        │ portable-pty│  │ Filesystem JSON │ │ sandbox-exec +   │
        │ (wezterm)   │  │ projects.json / │ │ CONNECT proxy    │
        │ agent CLIs  │  │ tasks/*.json /  │ │ (macOS Seatbelt) │
        │ git worktree│  │ settings.json   │ │                  │
        └────────────┘  └────────────────┘ └─────────────────┘
```

## Component Responsibilities

| Component | Responsibility | File |
|-----------|----------------|------|
| App shell / layout grid | Sidebar / main / right-panel column layout, settings overlay, focus restore | `src/App.tsx` |
| UnifiedBar | Window chrome, traffic-light spacing, breadcrumbs, sidebar toggle | `src/components/UnifiedBar.tsx` |
| Sidebar | Project + task tree, spotlight status, compact-mode hover reveal | `src/components/sidebar/Sidebar.tsx` |
| MainArea | Owns tab bar + active pane switching for the active task | `src/components/task/MainArea.tsx` |
| TerminalPane / AuxTerminal | xterm.js + WebGL rendering, wires to PTY IPC | `src/components/task/TerminalPane.tsx`, `src/components/task/AuxTerminal.tsx` |
| EditorPane | CodeMirror 6 editing surface for task worktree files | `src/components/task/EditorPane.tsx` |
| DiffPane / GitPanel | Git diff viewing, stage/unstage, commit | `src/components/task/DiffPane.tsx`, `src/components/task/GitPanel.tsx` |
| MarkdownPane / PreviewPane | Markdown preview + live preview URL iframe | `src/components/task/MarkdownPane.tsx`, `src/components/task/PreviewPane.tsx` |
| FileTree | Task worktree file browser, drag/drop, path operations | `src/components/task/FileTree.tsx` |
| RightPanel | Contextual panel host (diff/files/run/git) for active task | `src/components/task/RightPanel.tsx` |
| Settings | Preferences UI (agents, appearance, shortcuts, repo, prompts) | `src/components/settings/Settings.tsx` + `src/components/settings/*Section.tsx` |
| Dialogs | Modal dialog host (new task/project, command palette, confirm, etc.) | `src/components/dialogs/Dialogs.tsx` |
| App store | Central state: projects, tasks, tabs, split layout, spotlight | `src/store/app.ts` |
| Prefs store | Persisted user prefs (theme, fonts, shortcuts, sounds) | `src/store/prefs.ts` |
| UI store | Transient UI-only state (toasts, confirm dialogs, drag state) | `src/store/ui.ts` |
| ScriptRuns store | Per-(task, kind) streaming setup/run script lifecycle | `src/store/scriptRuns.ts` |
| IPC layer | Typed `invoke()`/`listen()` wrappers, one function per Rust command | `src/lib/ipc.ts` |
| Rust core | All Tauri commands: PTY spawn/write/resize, project/task CRUD, git ops, settings IO, script execution, spotlight polling | `src-tauri/src/lib.rs` |
| Sandbox | macOS `sandbox-exec` policy generation + monitoring for agent CLIs | `src-tauri/src/sandbox.rs` |
| Proxy | CONNECT-based network proxy enforcing sandbox host allowlist | `src-tauri/src/proxy.rs` |
| Repo config | Per-repo `.termic` config load/save/scaffold (allowed hosts/paths) | `src-tauri/src/repo_config.rs` |
| Shell env | Login-shell environment resolution for spawned PTYs | `src-tauri/src/shell_env.rs` |
| Automation | E2E/automation bridge commands (dev-only surface) | `src-tauri/src/automation.rs` |

## Pattern Overview

**Overall:** Thin desktop shell (Tauri 2 + WKWebView) over a React SPA, backed by a single-process Rust core acting as a local IPC command server. No client-server split, no database — flat JSON files on disk are the persistence layer.

**Key Characteristics:**
- Single Zustand store (`useApp`) is the source of truth for projects/tasks/tabs; components subscribe to narrow slices via selectors to avoid over-rendering.
- All privileged/OS-level work (PTY spawn, filesystem, git, sandboxing) lives exclusively in Rust; the frontend never touches `fs`/`child_process` directly — everything crosses through `src/lib/ipc.ts`.
- PTYs are process-lifetime: killed only on app quit or explicit task kill, never on component unmount (tabs can be closed/reopened without losing a running agent).
- No StrictMode (`src/main.tsx`) because double-invoking effects would race the async PTY spawn IPC call.
- Long-running/streaming operations (script runs, PTY output, spotlight status) use Tauri events (`listen()`) rather than polling.

## Layers

**Presentation (`src/components/`):**
- Purpose: Render UI, dispatch store actions, subscribe to store slices.
- Location: `src/components/task/`, `src/components/sidebar/`, `src/components/settings/`, `src/components/dialogs/`, `src/components/views/`, `src/components/ui/`
- Contains: React function components, mostly `.tsx`, colocated small hooks (e.g. `src/components/task/useTabStripDrag.ts`)
- Depends on: `src/store/*`, `src/lib/*`, `src/hooks/*`
- Used by: `src/App.tsx`

**State (`src/store/`):**
- Purpose: Own application data and transient UI state, expose actions that call into IPC and update state immutably.
- Location: `src/store/app.ts` (primary), `src/store/prefs.ts`, `src/store/ui.ts`, `src/store/scriptRuns.ts`, `src/store/update.ts`
- Contains: Zustand `create()` stores, no React-specific code
- Depends on: `src/lib/ipc.ts`, `src/lib/types.ts`, `src/lib/splitTree.ts`
- Used by: All components via `useApp(selector)` / `usePrefs(selector)` / etc.

**Domain/Utility logic (`src/lib/`):**
- Purpose: Pure/near-pure logic shared across components and stores — split-tree math, shortcut parsing, agent CLI detection, theming, markdown path resolution, IME handling.
- Location: `src/lib/*.ts`, `src/lib/explorer/`
- Contains: Framework-agnostic TS modules + colocated `*.test.ts` unit tests
- Depends on: `src/lib/ipc.ts` for a subset that needs the backend
- Used by: `src/store/*`, `src/components/*`

**IPC boundary (`src/lib/ipc.ts`):**
- Purpose: Single choke point translating Rust `#[tauri::command]`s into typed async functions; also wraps `listen()` for backend-pushed events.
- Location: `src/lib/ipc.ts`
- Contains: One exported function per Rust command, request/response types imported from `src/lib/types.ts`
- Depends on: `@tauri-apps/api`
- Used by: `src/store/*` (never called directly from components)

**Rust core (`src-tauri/src/`):**
- Purpose: Own all privileged operations — PTY lifecycle, project/task persistence, settings, git, script execution, sandbox policy, network proxy.
- Location: `src-tauri/src/lib.rs` (9,224 lines, all `#[tauri::command]` entry points + business logic) plus `sandbox.rs`, `proxy.rs`, `repo_config.rs`, `shell_env.rs`, `automation.rs`
- Contains: Tauri command handlers, `AppState`/managed state, JSON (de)serialization via `serde`
- Depends on: `portable-pty`, `tauri`, OS APIs (`sandbox-exec` on macOS)
- Used by: Frontend exclusively through `invoke()`

## Data Flow

### Primary Request Path (user runs an agent in a task)

1. User clicks a task in `src/components/sidebar/Sidebar.tsx` → `useApp.getState().setActiveTask(id)`
2. `MainArea` (`src/components/task/MainArea.tsx`) renders the active task's tab tree from `useApp` state (`splitTree`, `tabs`)
3. `TerminalPane` (`src/components/task/TerminalPane.tsx`) mounts, calls `ipc.ptySpawn(...)` (`src/lib/ipc.ts`)
4. Rust `pty_spawn` command (`src-tauri/src/lib.rs`) spawns the PTY via `portable-pty`, optionally wrapped by the sandbox (`sandbox.rs`) and proxy (`proxy.rs`) if the task's sandbox mode is enabled
5. PTY output streams back to the frontend via Tauri events; xterm.js renders it in `TerminalPane`
6. User keystrokes flow `TerminalPane` → `ipc.ptyWrite(...)` → `pty_write` command → PTY stdin

### Git / Diff Flow

1. `DiffPane`/`GitPanel` (`src/components/task/`) call `ipc.taskDiff` / `taskGitStatus` / `taskStage` / `taskCommit`
2. Rust commands (`task_diff`, `task_git_status`, `task_stage`, `task_commit` in `lib.rs`) shell out to `git` inside the task's worktree directory
3. Result JSON returns through IPC; store updates trigger re-render of diff/file lists

**State Management:**
- `useApp` (`src/store/app.ts`) holds normalized projects/tasks/tabs and split-tree layout; mutations always produce new objects (no direct mutation) per project immutability convention.
- `usePrefs` persists to `localStorage` and mirrors a subset to custom theme files on disk via IPC.
- `useUiStore` and `useScriptRuns` are intentionally separate stores so their high-churn updates (toasts, drag state, streaming script output) don't re-render subscribers of the much larger `useApp` store.

## Key Abstractions

**Task (worktree):**
- Purpose: Represents one git worktree branched off a project, with its own tabs (terminal/edit/diff), sandbox settings, and agent CLI session.
- Examples: `src/lib/types.ts` (`Task` type), `src-tauri/src/lib.rs` (`Task` struct + `task_*` commands)
- Pattern: One JSON file per task under `tasks/<uuid>.json`; frontend keeps a normalized in-memory copy in `useApp`.

**SplitTree / PaneLeaf:**
- Purpose: Recursive tree describing the split layout (terminal/edit/diff panes) within a task's main area.
- Examples: `src/lib/splitTree.ts`, consumed by `src/store/app.ts` and `src/components/task/SplitView.tsx`
- Pattern: Pure functional tree operations (find/replace/remove leaf) returning new trees, no mutation.

**IPC command pair:**
- Purpose: Each Rust `#[tauri::command]` has exactly one typed wrapper in `src/lib/ipc.ts`; frontend code never calls `invoke()` directly.
- Examples: `pty_spawn`/`ipc.ptySpawn`, `task_commit`/`ipc.taskCommit`
- Pattern: Command naming mirrors Rust `snake_case` function name → camelCase JS wrapper.

**Sandbox policy:**
- Purpose: Per-task opt-in `sandbox-exec` (Seatbelt) confinement for agent CLI PTYs, with an allowlist of hosts/paths enforced by a local CONNECT proxy.
- Examples: `src-tauri/src/sandbox.rs`, `src-tauri/src/proxy.rs`, `src-tauri/src/repo_config.rs`
- Pattern: Only the agent CLI PTY is sandboxed — AuxTerminal, setup/run/archive scripts are explicitly NOT sandboxed (see project CLAUDE.md).

## Entry Points

**Frontend bootstrap:**
- Location: `src/main.tsx`
- Triggers: WKWebView loads `index.html` → Vite-built bundle
- Responsibilities: Mounts `<App />` without `StrictMode` (documented async-race constraint), no other setup.

**App shell:**
- Location: `src/App.tsx`
- Triggers: Mount of the React tree
- Responsibilities: Loads all app data (`loadAll()`), wires window focus/blur handlers, registers global shortcuts (`useShortcuts`) and attention notifications (`useAttentionNotifier`), renders the 3-column grid layout + Settings overlay + Dialogs + Toaster.

**Rust entry point:**
- Location: `src-tauri/src/main.rs` (thin, delegates to `lib.rs`)
- Triggers: OS launches the compiled binary
- Responsibilities: Builds the Tauri `App`, registers the `generate_handler![...]` command table (in `lib.rs`), sets up managed state, plugins (window-state, etc.).

## Architectural Constraints

- **Threading:** Frontend is single-threaded (WKWebView JS runtime); IO-heavy Rust commands (PTY IO, git shellouts, script execution) must be async or spawned off the main thread — never block synchronously, or the whole webview event loop freezes (explicit project rule).
- **Global state:** Rust side uses Tauri-managed state (mutex-guarded structs) inside `lib.rs` for open PTYs, sandbox monitors, and spotlight pollers. Frontend global state is the Zustand stores (`useApp`, `usePrefs`, `useUiStore`, `useScriptRuns`) — all module-level singletons by design (Zustand's model).
- **No StrictMode:** `src/main.tsx` intentionally omits `React.StrictMode`; re-enabling would double-invoke effects and race the async PTY spawn IPC call.
- **Monolithic Rust module:** Nearly all backend logic lives in one file, `src-tauri/src/lib.rs` (9,224 lines) — it is not split by domain the way the frontend is; only sandboxing, proxying, repo-config, shell-env, and automation are factored into sibling modules.
- **CSP is intentionally narrow:** `tauri.conf.json` CSP should not be widened beyond the documented `img-src https:` exception (see `docs/sandbox.md` "known gap": the webview itself sits outside the sandbox).

## Anti-Patterns

### Synchronous IO-heavy Tauri commands

**What happens:** A `#[tauri::command]` that performs blocking filesystem, git, or process IO without `async`/thread offload.
**Why it's wrong:** WKWebView is bridged through a single event loop on macOS; a blocking command call freezes the entire UI (documented as a hard "don't" in project root CLAUDE.md).
**Do this instead:** Keep all task/project/script/sandbox commands async or dispatch to a background thread, following the existing pattern in `src-tauri/src/lib.rs`'s `task_*` and `pty_*` commands.

### Calling `invoke()` directly from components

**What happens:** Bypassing `src/lib/ipc.ts` and calling `@tauri-apps/api`'s `invoke()` inline in a component or store action.
**Why it's wrong:** Breaks the single choke point for typed request/response shapes and makes IPC payload changes hard to track across the codebase (see `docs/ipc.md` for the discipline this enforces).
**Do this instead:** Add/extend a wrapper function in `src/lib/ipc.ts` and import that.

### Sandboxing non-agent PTYs

**What happens:** Applying `sandbox-exec` confinement to AuxTerminal, setup/run/archive script PTYs.
**Why it's wrong:** The sandbox threat model only covers the agent CLI PTY; sandboxing other terminals breaks legitimate workflows without a corresponding scoped threat (explicit "don't" in root CLAUDE.md).
**Do this instead:** Only wrap the agent CLI spawn path (`pty_spawn` when the tab is the task's primary agent terminal) with sandbox policy from `sandbox.rs`.

## Error Handling

**Strategy:** Rust commands return `Result<T, String>` (or similar serializable error types) across the IPC boundary; the frontend treats IPC calls as fallible promises and surfaces failures via the toast system (`useUiStore`) or inline UI state.

**Patterns:**
- React error boundaries (`src/components/ErrorBoundary.tsx`) wrap major layout regions (Sidebar, MainArea, RightPanel, Settings) independently so one panel crashing doesn't blank the whole app.
- IPC call sites in `src/store/app.ts` typically `.catch()` and either toast an error or fail silently for best-effort background refreshes (e.g. spotlight resync).

## Cross-Cutting Concerns

**Logging:** Rust side uses `log_line` command plus stdout logging during dev; sandbox access/denial events are tracked in-memory (`sandbox_recent_access_*`/`sandbox_recent_denied_*` commands) and surfaced in the UI.
**Validation:** Rust command handlers validate paths/inputs before touching the filesystem or git (e.g. `path_exists`, `path_is_git_repo` checks precede risky operations).
**Authentication:** Not applicable — fully local desktop app, no network auth surface other than the sandboxed CONNECT proxy for agent CLI network egress.

---

*Architecture analysis: 2026-07-12*
