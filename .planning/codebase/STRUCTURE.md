# Codebase Structure

**Analysis Date:** 2026-07-12

## Directory Layout

```
termic/
├── src/                          # React 19 + TypeScript frontend
│   ├── main.tsx                  # Bootstrap (no StrictMode)
│   ├── App.tsx                   # App shell: layout grid, settings overlay, global effects
│   ├── index.css                 # Tailwind v4 @theme CSS vars, global styles
│   ├── lib/                      # Framework-agnostic logic, IPC, types, utils
│   │   ├── ipc.ts                # Typed wrappers around every Rust command
│   │   ├── types.ts              # Shared TS types (Project, Task, Tab, SplitTree, ...)
│   │   ├── splitTree.ts          # Pure split-pane tree operations
│   │   ├── agents.ts             # Agent CLI detection/config (claude/gemini/codex)
│   │   ├── shortcuts.ts          # Shortcut definitions/registry
│   │   ├── customTheme.ts        # Custom theme file parsing/caching
│   │   ├── editorTheme.ts        # CodeMirror theme mapping
│   │   ├── explorer/             # File-tree/explorer specific helpers
│   │   └── *.test.ts             # Co-located unit tests (vitest)
│   ├── store/                    # Zustand stores
│   │   ├── app.ts                # Primary store: projects, tasks, tabs, split layout
│   │   ├── prefs.ts               # Persisted user preferences (localStorage)
│   │   ├── ui.ts                  # Transient UI state (toasts, dialogs, drag)
│   │   ├── scriptRuns.ts          # Streaming setup/run script lifecycle
│   │   └── update.ts              # App update-checker state
│   ├── hooks/                    # Reusable React hooks
│   │   ├── useShortcuts.ts       # Global keyboard shortcut dispatch
│   │   ├── useAttentionNotifier.ts # OS notification / bounce-dock logic
│   │   └── useIsFullscreen.ts
│   ├── components/
│   │   ├── task/                 # Task-scoped UI: terminal, editor, diff, tabs, file tree
│   │   ├── sidebar/               # Project/task tree sidebar
│   │   ├── settings/               # Settings overlay sections
│   │   ├── dialogs/                # Modal dialog components
│   │   ├── views/                  # Top-level pages (Dashboard, History)
│   │   ├── ui/                     # Generic primitives (Button, Dialog, Dropdown, ...)
│   │   ├── UnifiedBar.tsx          # Window chrome / top bar
│   │   ├── ErrorBoundary.tsx       # Per-region error boundary
│   │   └── *.tsx                   # App-level shared components (SandboxIcon, etc.)
│   └── icons/                     # SVG/icon assets used by components
├── src-tauri/                     # Rust backend (Tauri 2)
│   ├── src/
│   │   ├── main.rs                # Thin binary entry, delegates to lib.rs
│   │   ├── lib.rs                 # ALL core logic: PTY, project/task IO, settings,
│   │   │                          #   scripts, git, sandbox orchestration, spotlight
│   │   ├── sandbox.rs             # sandbox-exec policy generation + monitoring
│   │   ├── proxy.rs               # CONNECT proxy enforcing sandbox allowlist
│   │   ├── repo_config.rs         # Per-repo .termic config load/save/scaffold
│   │   ├── shell_env.rs           # Login-shell environment resolution
│   │   └── automation.rs          # E2E/automation bridge commands
│   ├── capabilities/default.json  # Tauri permission capabilities
│   ├── examples/                  # Standalone smoke-test binaries (claude_smoke, pty_smoke)
│   ├── icons/                     # App icons (macOS/Android/iOS variants)
│   ├── assets/themes/             # Bundled built-in theme JSON files
│   ├── resources/sounds/          # Notification sound assets
│   └── tauri.conf.json            # Tauri app config, CSP, window settings
├── docs/                          # Architecture/reference docs (read when working in that area)
│   ├── ipc.md, data-model.md, tech-debt.md, performance.md, sandbox.md
│   ├── shortcuts.md, themes.md, ui.md, gotchas.md, automation.md
│   └── plans/docker-sandbox/      # Design docs for future work
├── scripts/                       # Repo-level build/release/utility scripts
├── public/                        # Static assets served by Vite
├── .claude/skills/                # Project-local skills (e2e, release)
├── .planning/                     # GSD planning artifacts (this document's home)
├── vite.config.ts                 # Vite config (dev port 1420, "@" -> src alias)
├── tsconfig.json                  # TS config, path alias "@/*" -> "./src/*"
└── package.json                   # npm scripts: tauri:dev, tauri:build, build
```

## Directory Purposes

**`src/lib/`:**
- Purpose: Framework-agnostic domain logic and the IPC boundary; the only layer allowed to talk to `@tauri-apps/api`.
- Contains: Pure TS utility modules, `ipc.ts`, `types.ts`, and one `.test.ts` per module with meaningful logic.
- Key files: `src/lib/ipc.ts` (IPC choke point), `src/lib/types.ts` (shared types, 29.8K), `src/lib/app.ts`-adjacent helpers used by the store.

**`src/store/`:**
- Purpose: Own all application and UI state via Zustand; the only place that both reads/writes state and calls IPC.
- Contains: `create()` store definitions, no JSX.
- Key files: `src/store/app.ts` (87.8K, primary store), `src/store/prefs.ts` (49.8K).

**`src/components/task/`:**
- Purpose: Everything rendered inside an active task's main/right panel — terminal, editor, diff, file tree, run controls, markdown preview.
- Contains: One component per pane/feature, plus small colocated hooks (`useTabStripDrag.ts`) and a test (`MarkdownPreview.test.ts`).
- Key files: `MainArea.tsx` (tab orchestration), `TerminalPane.tsx`/`AuxTerminal.tsx` (xterm.js), `EditorPane.tsx` (CodeMirror), `RightPanel.tsx`, `DiffPane.tsx`, `GitPanel.tsx`, `FileTree.tsx`.

**`src/components/sidebar/`:**
- Purpose: Project/task navigation tree, compact-mode reveal, update notification card.
- Key files: `Sidebar.tsx`, `ProjectActionsMenuItems.tsx`, `UpdateCard.tsx`.

**`src/components/settings/`:**
- Purpose: Settings overlay, split into per-section components.
- Key files: `Settings.tsx` (host), `AgentsSection.tsx`, `AppearanceSection.tsx`, `RepositorySection.tsx`, `ShortcutsSection.tsx`, `PromptLibrarySection.tsx`, `ExcludeEditor.tsx`, `GeneralSection.tsx`.

**`src/components/dialogs/`:**
- Purpose: Modal dialogs, all mounted via a single `Dialogs.tsx` host driven by `useUiStore`.
- Key files: `Dialogs.tsx` (host/router), `CommandPalette.tsx`, `NewTaskDialog.tsx`, `NewProjectDialog.tsx`, `FileFinderDialog.tsx`, `FindInFilesDialog.tsx`, `TaskSandboxDialog.tsx`.

**`src/components/ui/`:**
- Purpose: Generic, app-agnostic UI primitives built on Radix.
- Key files: `Dialog.tsx`, `Dropdown.tsx`, `Popover.tsx`, `ContextMenu.tsx`, `Button.tsx`, `Tooltip.tsx`, `Toaster.tsx`, `ResizeHandle.tsx`.

**`src-tauri/src/`:**
- Purpose: All Rust/native logic. `lib.rs` is intentionally monolithic (per project CLAUDE.md); only sandbox/proxy/repo-config/shell-env/automation are split out because they're independently testable subsystems.
- Key files: `lib.rs` (9,224 lines — command table, PTY, task/project IO, git, settings, scripts), `sandbox.rs` (113.2K), `proxy.rs` (26.8K), `shell_env.rs` (21.3K), `automation.rs` (16.8K), `repo_config.rs` (8.4K).

**`docs/`:**
- Purpose: Deep-reference docs for specific subsystems, meant to be read on-demand rather than upfront (per root CLAUDE.md's "read when working in that area" table).
- Generated: No. Committed: Yes.

## Key File Locations

**Entry Points:**
- `src/main.tsx`: Frontend bootstrap (mounts `<App/>`, no StrictMode)
- `src/App.tsx`: Layout shell, global effects, settings overlay
- `src-tauri/src/main.rs`: Rust binary entry
- `src-tauri/src/lib.rs`: Tauri `Builder` setup, `generate_handler![...]` command table (around line 8249)

**Configuration:**
- `vite.config.ts`: Dev server port 1420, `@` → `src` alias, Tailwind/React plugins
- `tsconfig.json`: `"@/*": ["./src/*"]` path alias
- `src-tauri/tauri.conf.json`: Window config, CSP (do not widen — see root CLAUDE.md)
- `src-tauri/capabilities/default.json`: Tauri permission capabilities

**Core Logic:**
- `src/lib/ipc.ts`: All frontend↔backend calls
- `src/store/app.ts`: Central task/project/tab state
- `src-tauri/src/lib.rs`: All backend command implementations

**Testing:**
- `src/lib/*.test.ts`: Vitest unit tests co-located with the module under test (e.g. `agents.test.ts`, `ime.test.ts`, `markdownPaths.test.ts`)
- `src/store/*.test.ts`: Store logic tests (`app.test.ts` 28.9K, `prefs.test.ts`, `update.test.ts`)
- `src/components/task/MarkdownPreview.test.ts`: Component-adjacent logic test
- `src-tauri/examples/*.rs`: Manual smoke-test binaries (not part of automated test suite)
- `.claude/skills/e2e/`: E2E/automation bridge skill (see `docs/automation.md`)

## Naming Conventions

**Files:**
- React components: `PascalCase.tsx` matching the exported component name (e.g. `TerminalPane.tsx` exports `TerminalPane`)
- Non-component TS modules: `camelCase.ts` (e.g. `splitTree.ts`, `customTheme.ts`)
- Tests: `<moduleName>.test.ts` co-located next to the module, or `*.integration.test.ts` for cross-module tests (e.g. `resume.integration.test.ts`)
- Rust modules: `snake_case.rs`, declared with `mod` in `lib.rs`

**Directories:**
- `src/components/<feature>/` groups by feature/region of the UI (task, sidebar, settings, dialogs), not by type
- `src/lib/`, `src/store/`, `src/hooks/` group by architectural layer

**IPC command naming:**
- Rust command: `snake_case` function name matching its `#[tauri::command]` (e.g. `task_diff`, `pty_spawn`)
- JS wrapper: camelCase in `src/lib/ipc.ts` (e.g. `taskDiff`, `ptySpawn`) — 1:1 mapping, always add both together

## Where to Add New Code

**New Feature (task-scoped UI, e.g. a new pane):**
- Component: `src/components/task/<FeatureName>.tsx`
- Wire into tab system via `src/lib/splitTree.ts` types and `src/store/app.ts` tab actions
- Tests: co-located `<FeatureName>.test.ts` if it contains non-trivial logic

**New Settings Section:**
- Component: `src/components/settings/<Name>Section.tsx`
- Register in `src/components/settings/Settings.tsx`
- Persisted values: extend `src/store/prefs.ts`

**New Dialog:**
- Component: `src/components/dialogs/<Name>Dialog.tsx`
- Register trigger/state in `src/store/ui.ts`, mount via `src/components/dialogs/Dialogs.tsx`

**New Rust Command:**
- Implementation: add `#[tauri::command]` fn in `src-tauri/src/lib.rs` (or the relevant sibling module — `sandbox.rs`/`proxy.rs`/`repo_config.rs`/`shell_env.rs`/`automation.rs` if domain-specific)
- Register in the `generate_handler![...]` list in `lib.rs`
- Add a matching typed wrapper in `src/lib/ipc.ts`
- Keep IO async — never block the main thread (see ARCHITECTURE.md anti-patterns)

**Shared Utilities:**
- Pure logic used by multiple components/stores: `src/lib/<name>.ts`
- Add a `src/lib/<name>.test.ts` alongside it for non-trivial logic

**Shared UI Primitives:**
- Generic Radix-based primitives: `src/components/ui/<Name>.tsx`

## Special Directories

**`.planning/`:**
- Purpose: GSD workflow planning artifacts (phases, plans, codebase maps — this document lives here)
- Generated: Partially (mapper/planner agents write here)
- Committed: Yes

**`src-tauri/target/`:**
- Purpose: Rust build output
- Generated: Yes
- Committed: No

**`dist/` (Vite build output, not present until build):**
- Purpose: Frontend production bundle consumed by Tauri bundler
- Generated: Yes
- Committed: No

**`src-tauri/target/release/bundle/`:**
- Purpose: Final `.app`/`.dmg` output of `npm run tauri:build`
- Generated: Yes
- Committed: No

**`docs/plans/docker-sandbox/`:**
- Purpose: Design docs for a not-yet-implemented sandboxing approach
- Generated: No
- Committed: Yes

---

*Structure analysis: 2026-07-12*
