# Testing Patterns

**Analysis Date:** 2026-07-12

## Test Framework

**Runner:**
- Vitest 3.x (`vitest` in `devDependencies`, `package.json`)
- No standalone `vitest.config.ts` — Vitest is invoked directly via `vitest run` and picks up defaults plus whatever the Vite plugin config in `vite.config.ts` provides implicitly (React + Tailwind plugins, `@` alias). Per-file environment overrides are used instead of a global `environment` setting (see below).

**Assertion Library:**
- Vitest's built-in `expect` (Jest-compatible API) — no Chai/Jest imported separately

**DOM Environment:**
- `happy-dom` (`devDependencies`) used per-file via the magic comment `// @vitest-environment happy-dom` at the top of DOM-touching tests, e.g. `src/store/resume.integration.test.ts:1`. Pure-logic tests (no DOM) omit this and run in Node's default environment.

**Run Commands:**
```bash
npm run test           # vitest run (single pass, all *.test.ts files)
npm run test:signals   # node scripts/dev.signals.test.mjs — separate signal-handling test for the dev script, run directly with node (not Vitest)
```
No `--watch` or `--coverage` script is defined in `package.json`; run Vitest directly with `npx vitest --coverage` or `npx vitest watch` if needed ad hoc.

**Rust tests:**
```bash
cargo test --manifest-path src-tauri/Cargo.toml
```

## Test File Organization

**Location:**
- Co-located with the module under test, same directory, same base name: `src/lib/utils.ts` → `src/lib/utils.test.ts`. No `__tests__/` or `test/` directory convention.

**Naming:**
- `<module>.test.ts` for unit tests
- `<module>.integration.test.ts` for tests that exercise multiple real modules together instead of mocking them (e.g. `src/store/resume.integration.test.ts`)

**Files present (frontend):**
- `src/lib/agents.test.ts`
- `src/lib/utils.test.ts`
- `src/lib/terminalTitle.test.ts`
- `src/lib/markdownPaths.test.ts`
- `src/lib/ime.test.ts`
- `src/lib/loginShell.test.ts`
- `src/lib/customTheme.test.ts`
- `src/lib/archiveTask.test.ts`
- `src/lib/previewPaths.test.ts`
- `src/store/app.test.ts`
- `src/store/resume.integration.test.ts`
- `src/store/update.test.ts`
- `src/store/prefs.test.ts`
- `src/components/task/MarkdownPreview.test.ts`

Coverage is concentrated in `src/lib/` (pure logic) and `src/store/` (state machines); component-level tests exist but are the minority (only `MarkdownPreview.test.ts` under `src/components/`).

**Rust tests:**
- Co-located in the same file, bottom of module, inside `#[cfg(test)] mod tests { use super::*; ... }`. Present in `src-tauri/src/shell_env.rs`, `src-tauri/src/proxy.rs`, `src-tauri/src/lib.rs`, `src-tauri/src/repo_config.rs`, `src-tauri/src/sandbox.rs`.

## Test Structure

**Suite Organization (TS/Vitest):**
```typescript
import { describe, it, expect } from "vitest";
import { slugify, branchify, shortPath } from "@/lib/utils";

// ── slugify ───────────────────────────────────────────────────────────

describe("slugify", () => {
  it("lowercases and replaces spaces with hyphens", () => {
    expect(slugify("Hello World")).toBe("hello-world");
  });
  ...
});
```
(`src/lib/utils.test.ts`)

**Patterns:**
- One `describe` block per exported function/behavior; box-drawing comment banners (`// ── name ───...`) separate sections within a file when testing multiple exports
- `it(...)` descriptions are full behavioral sentences: `"strips leading and trailing hyphens"`, `"collapses multiple consecutive non-alphanum chars into one hyphen"` — read as a spec, not just a label
- `beforeEach` used to reset shared/global store state before each test in store tests, e.g. `useApp.setState({ tabs: {}, activeTab: {}, activeTaskId: null, ... })` (`src/store/resume.integration.test.ts`)
- Inline comments inside tests explain *why* an input produces a given output when non-obvious (unicode handling, regression case numbers)

**Rust:**
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn fallback_path_adds_homebrew_when_missing() {
        let result = fallback_path("/usr/bin:/bin");
        assert!(result.contains("/opt/homebrew/bin"), "must add homebrew bin");
    }
}
```
(`src-tauri/src/shell_env.rs:340`)

## Mocking

**Framework:** Vitest's built-in `vi` (`vi.mock`, `vi.fn`, `vi.spyOn`, `vi.mocked`) — no separate mocking library (no `sinon`, no `jest-mock`).

**Patterns:**
```typescript
vi.mock("@/lib/ipc", () => ({
  ptyKill: vi.fn().mockResolvedValue(undefined),
  taskSetTabs: vi.fn().mockResolvedValue(undefined),
  taskSetTabSessionId: vi.fn().mockResolvedValue(undefined),
}));
vi.mock("@/lib/tabFocus", () => ({ focusTerminalTab: vi.fn() }));
```
(`src/store/resume.integration.test.ts`)

```typescript
const readBase64 = vi.mocked(taskFileReadBase64);
const nowSpy = vi.spyOn(Date, "now").mockReturnValue(1_000_000);
```
(`src/components/task/MarkdownPreview.test.ts`)

Hoisted-mock pattern when a mock factory needs to reference shared fns declared outside it:
```typescript
// Hoisted mocks so the vi.mock factories can reference them.
const { taskArchive, loadAll, setActiveTask } = ...
```
(`src/lib/archiveTask.test.ts`)

**What to Mock:**
- Always mock `@/lib/ipc` (the Tauri `invoke` boundary) — tests never make real Tauri IPC calls
- Mock cross-cutting stores (`@/store/app`, `@/store/ui`) when the module under test only reads a narrow slice of them and the rest of the store isn't relevant
- Mock focus/DOM side-effect helpers (`@/lib/tabFocus`) that have no meaningful assertion value in a unit test

**What NOT to Mock:**
- Pure logic modules under test stay real, even when combined with other real modules in integration tests — `resume.integration.test.ts` explicitly keeps `useApp`, `decideResume`, and `spawnArgsForCli` real, mocking only `ipc`/`tabFocus`, per its file-header comment: "using the REAL store, the REAL decideResume, and the REAL spawnArgsForCli"
- Don't mock the module you're directly testing

## Fixtures and Factories

**Test Data:**
Local factory functions defined per test file, not shared across files:
```typescript
function makeTask(o: Partial<Task> = {}): Task {
  return {
    id: "ws1", project_id: "p1", name: "seo improvements", branch: "main",
    base_branch: "main", path: "/x/ws1", cli: "claude", port: 1420,
    created: "2024-01-01", archived: false, ...o,
  } as Task;
}
```
(`src/store/resume.integration.test.ts`) — takes a `Partial<T>` override object, spread over sane defaults.

**Location:**
- No shared `fixtures/` or `factories/` directory — each test file defines its own minimal factories/builders local to that file's needs.

## Coverage

**Requirements:** No coverage threshold configured (no `coverage` block in a Vitest config, no CI coverage gate found in this exploration).

**View Coverage:**
```bash
npx vitest run --coverage
```
(ad hoc; not wired into `package.json` scripts)

## Test Types

**Unit Tests:**
- Primary test type in this repo — pure function behavior in `src/lib/*.test.ts`, store action behavior in `src/store/*.test.ts`

**Integration Tests:**
- `*.integration.test.ts` naming convention for tests that wire multiple real modules together (store + business logic) while mocking only the IPC/OS boundary. See `src/store/resume.integration.test.ts` for the canonical example and its rationale comment.

**E2E Tests:**
- Not part of the automated test suite. Live/manual E2E testing exists via the project's `e2e` skill and automation bridge (see `docs/automation.md`), but per project `CLAUDE.md`: do not run it proactively — only when explicitly requested or necessary, since it drives the real app.

**Rust unit tests:**
- Colocated `#[cfg(test)]` modules test pure helper functions (path fallback logic, config normalization, sandbox path handling) — no integration-level Rust test harness observed.

## Common Patterns

**Async / Store Reset:**
```typescript
beforeEach(() => {
  useApp.setState({
    tabs: {}, activeTab: {}, activeTaskId: null,
    ...
  });
});
```
Reset global Zustand store state before each test to avoid cross-test leakage (stores are module-level singletons).

**Time-dependent Testing:**
```typescript
const nowSpy = vi.spyOn(Date, "now").mockReturnValue(1_000_000);
```
Freeze `Date.now()` via `vi.spyOn` for deterministic timestamp assertions (`src/components/task/MarkdownPreview.test.ts`).

**DOM-environment opt-in:**
```typescript
// @vitest-environment happy-dom
```
Add this magic comment at the top of any test file that touches DOM APIs; omit it for pure-logic tests to keep them fast and environment-independent.

---

*Testing analysis: 2026-07-12*
