# Codebase Concerns

**Analysis Date:** 2026-07-12

## Tech Debt

**`workspace` → `task` migration (schema v1) — tracked, not yet purgeable:**
- Issue: ~320 LOC of one-time on-disk migration scaffolding (`migrate_workspaces_to_tasks()`, `MigrationLock`, `stamp_schema_version`, `repoint_task_bases()`, serde aliases, `src/lib/lsMigration.ts`) kept alive to upgrade pre-rename (`v0.19.0`) profiles.
- Files: `src-tauri/src/lib.rs` (search for `migrate_workspaces_to_tasks`, `MigrationLock`, `repoint_task_bases`), `src/lib/lsMigration.ts`, `src/main.tsx` (import).
- Impact: none currently (idempotent, gated behind `schema_version >= 1`), but it's dead weight indefinitely until purged.
- Fix approach: full purge checklist already documented in `docs/tech-debt.md`; execute a few minor releases after v0.19 once no active install predates the rename.

**`migrate_legacy_members()` (multi-repo) — untracked lifetime:**
- Issue: separate, older load-time migration in `load_projects()`, not gated by `schema_version`, so its removal criterion is independent and easy to forget.
- Files: `src-tauri/src/lib.rs` (search `migrate_legacy_members`).
- Impact: none currently; same "silently permanent" risk as any ungated migration.
- Fix approach: per `docs/tech-debt.md` #2, assess separately and remove once no active profile predates multi-repo members.

**`lib.rs` is a 9,224-line God file:**
- Issue: nearly all Rust backend logic — PTY spawn/lifecycle, project/task persistence, settings, scripts, git, sandbox wiring, migrations, IPC commands — lives in one file (`src-tauri/src/lib.rs`, 410 KB). This is explicitly called out in `CLAUDE.md` ("ALL Rust ... in `src-tauri/src/lib.rs`") as intentional today, but it directly violates the "many small files" convention in the user's own coding-style rules (200-400 lines typical, 800 max).
- Files: `src-tauri/src/lib.rs`.
- Impact: every Rust change risks touching unrelated logic; code review and blame history are hard to scope; compile times for any edit are whole-crate; onboarding cost is high.
- Fix approach: incrementally split by domain (PTY/terminal, project/task CRUD, settings, scripts, git ops, process-tree kill helpers) into separate modules under `src-tauri/src/`, following the pattern already used for `sandbox.rs`, `proxy.rs`, `shell_env.rs`, `automation.rs`, `repo_config.rs`. Do this opportunistically alongside feature work, not as a big-bang rewrite (risk of regressions in PTY lifecycle code is high).

**`sandbox.rs` is a 2,478-line single file:**
- Issue: second-largest file in the repo, holds the sandbox-exec profile generation, CONNECT proxy interaction, and policy logic together.
- Files: `src-tauri/src/sandbox.rs`.
- Impact: same maintainability risk as `lib.rs`, smaller scale. Multiple `#[allow(dead_code)]` blocks (lines 59, 70, 1394, 1537, 1769) suggest speculative/unused code paths accumulating.
- Fix approach: audit the `#[allow(dead_code)]` items — either wire them up or delete; consider splitting profile-generation from policy/session-state management.

**Frontend god components:**
- Issue: several React components exceed the 800-line hard cap from coding-style rules: `src/components/task/TerminalPane.tsx` (2,776 lines), `src/lib/explorer/fileIcons.ts` (2,681 — data table, less concerning), `src/components/sidebar/Sidebar.tsx` (1,703), `src/components/task/MarkdownPreview.tsx` (1,134), `src/components/settings/RepositorySection.tsx` (1,092), `src/components/task/GitPanel.tsx` (992), `src/store/prefs.ts` (983), `src/components/dialogs/NewTaskDialog.tsx` (971), `src/components/settings/AgentsSection.tsx` (955), `src/components/task/RightPanel.tsx` (936).
- Files: as listed above.
- Impact: `TerminalPane.tsx` in particular mixes PTY lifecycle, xterm setup, font-loading gates (see gotchas), and rendering in one file — the exact area the project's own docs flag as most flicker/regression-sensitive.
- Fix approach: extract pure helpers (font gating, PTY payload shaping, resize/fit logic) into `src/lib/` modules with unit tests, leaving the component focused on render + wiring. Do incrementally; this code has proven timing-sensitive bugs (see gotchas), so refactors need care and manual re-verification in the real app, not just type-checking.

**High `.clone()` usage in `lib.rs` (196 occurrences):**
- Issue: pervasive cloning in the largest Rust file, some likely necessary (Tauri command handlers need owned data to cross async boundaries) but the volume suggests possible unnecessary allocation/ownership churn.
- Files: `src-tauri/src/lib.rs`.
- Impact: minor runtime cost; more importantly, a signal of unclear ownership boundaries that would surface during any `lib.rs` split.
- Fix approach: no urgent action; revisit clone sites when splitting `lib.rs` into modules — many will naturally resolve via `Arc`/reference passing.

**`console.log` debug statements left in shipped code:**
- Issue: several unconditional `console.log` calls outside test files.
- Files: `src/components/task/TerminalPane.tsx:637-638` (behind a debug helper, but the helper itself has no build-time strip), `src/components/task/TerminalPane.tsx:1245` (`ptyDebug` logging path), `src/lib/terminalRenderer.ts:17-18` (WebGL renderer diagnostics), `src/store/update.ts:348` (mock-updater no-op log, intentional dev-only path).
- Impact: low — mostly gated behind debug flags — but `terminalRenderer.ts:17-18` runs unconditionally on the renderer-creation failure path and could spam the console in production.
- Fix approach: gate all remaining unconditional `console.log` calls behind a dev/debug flag consistent with the `ptyDebug` pattern already used in `TerminalPane.tsx`.

## Known Bugs

No open, reproducible bugs are tracked directly in code (no `FIXME`/`BUG` markers found via `grep -rn "TODO\|FIXME\|HACK\|XXX"` across `src/` and `src-tauri/src/` — zero hits). All previously-encountered bugs are documented as *fixed*, with root cause and guard rails, in `docs/gotchas.md`. Treat these as regression traps rather than open bugs — the fix is already in code, but the surrounding pattern is fragile and easy to reintroduce:

**Terminal glyph-atlas corruption on PTY-output race (GH #70), fixed:**
- Symptoms: mixed letter heights in a terminal; selecting text visually "fixes" already-rendered rows.
- Files: `src/store/prefs.ts` (font warm-load), `src/lib/terminalRenderer.ts` (`awaitTerminalFonts()`), `src/components/task/TerminalPane.tsx` and `src/components/task/AuxTerminal.tsx` (gate first fit + PTY spawn).
- Trigger: any new pane that rasterizes text with a bundled font and skips the `awaitTerminalFonts()` gate will reintroduce this.
- Workaround: none needed if the existing gate is respected; do not "fix" future recurrences with a naive `clearTextureAtlas()+fit()` (documented as unsafe — can race PTY spawn or fire against a collapsed 0x0 host).

**Detached-canvas font resolution failure in WKWebView:**
- Symptoms: a `<canvas>` never attached to the document silently falls back to a system font for any user-installed font (e.g. `~/Library/Fonts`), even for plain ASCII, with no error signal — `document.fonts.check()` also lies (always returns true in WebKit).
- Files: any font-probe helper (see `docs/gotchas.md` for detection technique: rasterize + compare pixel signatures against a known-missing font).
- Trigger: probing font availability from a detached canvas.
- Workaround: attach canvas before measuring; xterm itself is unaffected since its canvas is already attached.

## Security Considerations

**Sandbox scope is process/CLI-only, webview is explicitly outside it:**
- Risk: `docs/sandbox.md` documents this as a "Known gap" — the WKWebView (rendering the whole app UI) sits outside the `sandbox-exec` + CONNECT proxy sandbox applied to agent CLI PTYs. A compromised renderer (e.g. via a malicious markdown preview or malicious remote image) is not contained by the sandbox.
- Files: `src-tauri/src/sandbox.rs`, `src-tauri/src/proxy.rs`, `tauri.conf.json` (CSP).
- Current mitigation: CSP is deliberately kept tight (`CLAUDE.md` forbids widening it; only `img-src https:` is an accepted exception), and remote images in markdown preview are gated behind a default-off preference (per recent commit `98e5362`).
- Recommendations: no code action needed beyond continued CSP discipline; this is an accepted, documented architectural gap rather than an oversight.

**Only the agent CLI PTY is sandboxed by design; other scripts run unsandboxed:**
- Risk: per `CLAUDE.md` explicit instruction, AuxTerminal, setup/run/archive scripts are intentionally NOT sandboxed — only the primary agent CLI PTY is, matching the documented threat model (agent output is the untrusted surface, not user-authored scripts).
- Files: `src-tauri/src/sandbox.rs`, `src-tauri/src/lib.rs` (`task_set_sandbox` and script execution paths).
- Current mitigation: `task_set_sandbox` is required to SIGKILL live PTYs by default before changing sandbox state (per `CLAUDE.md` "what not to do" list) — verify this invariant holds at every call site if touching sandbox toggling code.
- Recommendations: any future change that adds new script-execution surfaces should explicitly decide sandbox status rather than defaulting to "unsandboxed," to avoid silently expanding the threat model.

**Extensive `unsafe { libc::kill(...) }` process-group termination logic:**
- Risk: 16 `unsafe` blocks in `src-tauri/src/lib.rs` (plus 2 in `automation.rs`, 1 in `shell_env.rs`) directly call `libc::kill` with process-group-negated PIDs (`-pid`) to terminate PTY process trees on task close/archive/sandbox-toggle. Passing a wrong or stale PID/PGID to a negated `kill()` can signal an unrelated process group.
- Files: `src-tauri/src/lib.rs` (lines ~1629, 3836, 3994, 6138, 6532-6534, 6624, 6687, 6770, 6807, 8324-8460), `src-tauri/src/automation.rs:246,259`, `src-tauri/src/shell_env.rs:97`.
- Current mitigation: none visible beyond care at each call site (no wrapping "safe kill" helper); this logic is duplicated ~10+ times across the file rather than centralized.
- Recommendations: extract a single `kill_process_group(pid, signal)` safe wrapper with PID-liveness re-check before signaling, to reduce duplication and the chance of a copy-paste PID/PGID mixup in future edits. High priority given how central "don't leave orphan PTYs" is to app correctness.

**166 `.unwrap()` calls in `lib.rs`, 8 in `repo_config.rs`, 5 in `sandbox.rs`, 4 in `automation.rs`:**
- Risk: an `.unwrap()` on a `Result`/`Option` that turns out to be `Err`/`None` at runtime panics the async task or (depending on Tauri's panic behavior) can crash the whole process, directly contradicting the "no user-visible crash" expectation of a desktop app doing IO-heavy work (file/PTY/git operations).
- Files: `src-tauri/src/lib.rs` (166), `src-tauri/src/repo_config.rs` (8), `src-tauri/src/sandbox.rs` (5), `src-tauri/src/automation.rs` (4).
- Current mitigation: unknown without per-call audit — many are likely on infallible operations (e.g. `Mutex::lock()` on a never-poisoned mutex, regex compiled from a literal). No systematic audit exists.
- Recommendations: audit `.unwrap()` calls that touch fallible IO (file reads/writes, JSON parse, external command spawn) and convert to `?`/explicit error returns with `tauri::Error` or a typed app error, especially any reachable from user-controlled input (task names, file paths, YAML script content).

## Performance Bottlenecks

**No dedicated performance-profiling harness found:**
- Problem: the project's own `CLAUDE.md` states "Performance trumps polish" as a core value (flicker, >100ms editor open, unnecessary sidebar re-renders are "real regressions"), and `docs/performance.md` exists as a reference, but there is no automated perf regression test (e.g. render-count assertions, startup-time budget test) in the test suite (`src/store/app.test.ts`, `src/store/*.test.ts` are logic/state tests, not perf tests).
- Files: `docs/performance.md` (guidance only), no perf test files found under `src/`.
- Cause: perf regressions are currently caught only by manual testing/dogfooding, not CI.
- Improvement path: consider a lightweight render-count assertion (React Testing Library + a render-spy) for the highest-risk components (`Sidebar.tsx`, `TerminalPane.tsx`, `RightPanel.tsx`) to catch selector-identity regressions (a documented Zustand trap, see below) before they ship.

**Zustand selector identity traps (documented, self-monitored):**
- Problem: `docs/gotchas.md` explicitly warns against returning new objects/arrays from selectors without memoization, and against using object identity (`ws`, `tab`) in effect deps instead of stable IDs. `src/store/app.ts` is 1,892 lines and `src/store/prefs.ts` is 983 lines — large surface area for this class of bug to reappear as the store grows.
- Files: `src/store/app.ts`, `src/store/prefs.ts`, `src/store/ui.ts`, `src/store/scriptRuns.ts`.
- Cause: large, actively-growing Zustand stores with many derived selectors used across the whole component tree.
- Improvement path: no code change needed proactively; when adding new selectors, follow the existing frozen-constant-default pattern already established in the store files.

## Fragile Areas

**Font-loading / glyph-atlas gating in terminal panes:**
- Files: `src/lib/terminalRenderer.ts` (`awaitTerminalFonts()`), `src/store/prefs.ts` (font warm-load), `src/components/task/TerminalPane.tsx`, `src/components/task/AuxTerminal.tsx`.
- Why fragile: the fix for GH #70 depends on strict ordering (warm font load at module start → gate first fit + PTY spawn on font readiness, capped at 800ms) — a subtle, timing-dependent invariant that is easy to violate when adding any new terminal-rendering pane or touching PTY spawn sequencing. `docs/gotchas.md` explicitly warns future panes must go through the same gate.
- Safe modification: any new component that rasterizes text via a bundled font must call `awaitTerminalFonts()` (or equivalent) before its first fit/spawn; never revert to a post-hoc `clearTextureAtlas()+fit()` approach.
- Test coverage: no automated test exercises the font-race scenario (inherently a timing/rendering integration issue, hard to unit test) — relies entirely on manual verification and the documented gotcha.

**PTY payload shape and IPC command contracts:**
- Files: `src/lib/ipc.ts` (598 lines), `src-tauri/src/lib.rs` (Tauri command handlers), `docs/ipc.md`.
- Why fragile: `docs/gotchas.md` records a specific footgun — `pty_spawn` "invalid length 0" from forgetting to wrap `SpawnArgs` in `{ args: ... }` — indicating the IPC payload contract is not type-enforced across the Rust/TS boundary (no shared schema/codegen visible), relying on manual shape-matching.
- Safe modification: cross-reference `docs/ipc.md` before changing any command signature; consider whether new commands should follow an existing wrapped-args convention to avoid recreating this bug.
- Test coverage: no IPC contract tests found; risk is caught only at runtime.

**Process-tree termination logic (kill/SIGTERM/SIGKILL sequences) scattered across `lib.rs`:**
- Files: `src-tauri/src/lib.rs` (10+ separate `unsafe { libc::kill(...) }` sites, see Security section above).
- Why fragile: duplicated rather than centralized; each site independently decides PID vs. process-group targeting and signal choice (SIGTERM then SIGKILL escalation appears in some but not all paths, e.g. lines 6532-6534 show a liveness recheck loop that other sites lack).
- Safe modification: when touching task close/archive/sandbox-toggle flows, verify the existing kill sequence at that specific call site rather than assuming consistency across the file; do not copy a pattern from one site to another without confirming they need the same escalation behavior.
- Test coverage: `src-tauri/src/lib.rs` has 43 `#[test]` functions but process-kill logic (which requires spawning real child processes) is inherently hard to unit test — likely under-covered relative to its criticality.

**`TerminalPane.tsx` (2,776 lines) mixes multiple concerns:**
- Files: `src/components/task/TerminalPane.tsx`.
- Why fragile: combines PTY lifecycle (spawn/resize/kill), xterm.js + WebGL addon setup (with its own documented dispose-ordering bug — see gotchas: "Dispose `webglAddon` BEFORE `term.dispose()`"), font-loading gates, and debug logging in one file with no internal module boundaries.
- Safe modification: prefer additive changes; before any refactor, re-read `docs/gotchas.md` in full (WebGL dispose order, font race, StrictMode-off rationale) since this file is where most of those historical bugs originated.
- Test coverage: zero dedicated test file for `TerminalPane.tsx` (only 1 test file exists under `src/components/task/` — `MarkdownPreview.test.ts` — out of 22 `.tsx` files in that directory).

**React StrictMode permanently disabled:**
- Files: `src/main.tsx`.
- Why fragile: `CLAUDE.md` states double-invoke races the async PTY spawn, and explicitly instructs not to re-enable without auditing every async effect's cleanup. This means the app cannot benefit from StrictMode's double-invoke bug detection for effects going forward, and any new effect that isn't cancellation-safe won't be caught by this class of tooling.
- Safe modification: new `useEffect` hooks with async setup must implement explicit cleanup/cancellation manually (per the React/Zustand traps section in `docs/gotchas.md`) since there's no automated safety net.
- Test coverage: not applicable (a tooling/dev-mode gap, not a runtime bug).

## Scaling Limits

Not applicable — this is a single-user, on-device desktop app (`CLAUDE.md`: "app is entirely on-device," no server/backend daemon). No multi-user, database-scale, or request-volume concerns apply. The relevant "scaling" axis is number of concurrent tasks/PTYs per user session, which is not currently documented with an explicit cap; the process-tree kill logic (see Fragile Areas) is the most likely place a large number of concurrent tasks would surface bugs.

## Dependencies at Risk

**CodeMirror 6 pinned as the only viable editor (Monaco explicitly rejected):**
- Risk: `CLAUDE.md` states Monaco was "verified slower in WKWebView" and instructs not to swap editors without asking. This isn't a risk in the traditional sense (no known CVE or abandonment) but is a hard architectural constraint that limits editor-feature options to whatever CodeMirror 6's ecosystem supports.
- Impact: any future editor feature request that assumes Monaco-specific APIs will need a CodeMirror-native reimplementation.
- Migration plan: none needed; this is an intentional, tested decision, not debt.

No abandoned, unmaintained, or CVE-flagged dependencies were identified in this pass (package manifest was not deeply audited for CVEs — recommend running `npm audit` and `cargo audit` periodically as a separate check outside this structural review).

## Missing Critical Features

Not applicable to this pass — no missing-feature gaps were identified as "concerns"; feature scope questions belong in `docs/tech-debt.md` / roadmap docs, not this technical-debt audit.

## Test Coverage Gaps

**Rust backend: 43 tests in `lib.rs` (9,224 lines) vs. much deeper coverage in smaller files:**
- What's not tested: the core `lib.rs` file has the lowest test-density (43 tests / 9,224 lines) relative to `sandbox.rs` (57 tests / 2,478 lines) and `shell_env.rs` (24 tests / 542 lines), both of which are more thoroughly covered per line.
- Files: `src-tauri/src/lib.rs`.
- Risk: PTY lifecycle, migration logic, and project/task CRUD (the largest and most user-facing surfaces in the backend) are proportionally under-tested relative to sandbox/shell-env code.
- Priority: High — this file also has the highest `.unwrap()` count (166) and the most `unsafe` blocks (16), compounding the risk of untested panics/crashes.

**Frontend: only 14 test files total, concentrated in `src/lib/` and `src/store/`:**
- What's not tested: `src/components/` has only 1 test file (`MarkdownPreview.test.ts`) across 6 component subdirectories (`task/`, `sidebar/`, `settings/`, `dialogs/`, `ui/`, `views/`). None of `Sidebar.tsx` (1,703 lines), `TerminalPane.tsx` (2,776 lines), `GitPanel.tsx` (992 lines), `RightPanel.tsx` (936 lines), `NewTaskDialog.tsx` (971 lines), `AgentsSection.tsx` (955 lines), or `UnifiedBar.tsx` (617 lines) have dedicated tests.
- Files: `src/components/**/*.tsx` (21 of 22 files under `src/components/task/` alone are untested).
- Risk: the highest-risk, highest-LOC UI logic (terminal lifecycle, git operations, file tree/sidebar rendering) has no regression safety net beyond manual verification; this matches the "Performance trumps polish" area the project cares most about, yet it's the least tested.
- Priority: High for `TerminalPane.tsx` and `GitPanel.tsx` given their history of subtle, hard-to-reproduce bugs documented in `docs/gotchas.md`; Medium for the rest.

**No IPC/contract-boundary tests between Rust and TypeScript:**
- What's not tested: the Tauri command payload shapes (`src/lib/ipc.ts` ↔ `src-tauri/src/lib.rs` command handlers) have no automated contract test, despite a documented historical bug (`pty_spawn` "invalid length 0") caused by a payload-shape mismatch.
- Files: `src/lib/ipc.ts`, `src-tauri/src/lib.rs`.
- Risk: any future IPC signature change can silently break at runtime with no compile-time or test-time signal, since TS and Rust types are maintained independently (no shared schema generation observed).
- Priority: Medium — the app is not IPC-heavy enough to warrant a full contract-testing framework, but a lightweight smoke test per command (call with real args, assert no error) would catch shape drift.

---

*Concerns audit: 2026-07-12*
