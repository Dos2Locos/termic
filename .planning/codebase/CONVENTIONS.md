# Coding Conventions

**Analysis Date:** 2026-07-12

## Naming Patterns

**Files:**
- React components: `PascalCase.tsx` (e.g. `src/components/task/TabBar.tsx`, `src/components/task/MarkdownPreview.tsx`)
- Plain TS modules/utilities: `camelCase.ts` (e.g. `src/lib/utils.ts`, `src/lib/terminalTitle.ts`, `src/lib/customTheme.ts`)
- Zustand stores: `camelCase.ts` under `src/store/` (`app.ts`, `ui.ts`, `prefs.ts`, `scriptRuns.ts`)
- Test files: co-located `<name>.test.ts` next to the module under test (never a separate `__tests__/` tree)
- Rust: `snake_case.rs` module files in `src-tauri/src/` (`shell_env.rs`, `repo_config.rs`, `sandbox.rs`); most Tauri commands live in the single `src-tauri/src/lib.rs` (9200+ lines)

**Functions:**
- `camelCase`, descriptive verbs: `slugify`, `branchify`, `cleanLines`, `shortPath` (`src/lib/utils.ts`)
- Rust: `snake_case` fn names, `Result<T>` return type alias from `anyhow::Result` (`src-tauri/src/lib.rs:1105` `fn ... -> Result<String>`)

**Variables:**
- `camelCase` for locals/state
- `SCREAMING_SNAKE_CASE` constants for localStorage keys and fixed config, grouped at top of module: `LS_EDITOR_FONT`, `LS_THEME`, `LS_SHORTCUTS` (`src/store/prefs.ts:27-53`)

**Types:**
- `PascalCase` for types/interfaces: `Task`, `TerminalTab`, `Agent`, `CustomTheme` (`src/lib/types.ts`, `src/lib/customTheme.ts`)
- Union string-literal types for modes/enums: `export type MarkdownView = "source" | "preview" | "split";` (`src/store/prefs.ts:56`)
- Template literal types for extensible ids: `type ThemeMode = BuiltinThemeMode | \`custom:${string}\`;` (`src/store/prefs.ts:61`)

## Code Style

**Formatting:**
- No Prettier config detected; formatting is implicit/manual (2-space indent, double quotes, semicolons used throughout `.ts`/`.tsx`)
- Single-line function bodies used freely for one-liners: `export function cn(...inputs: ClassValue[]) { return twMerge(clsx(inputs)); }` (`src/lib/utils.ts:5`)
- Comment banners with box-drawing chars used to delimit test/logic sections: `// ── slugify ───────────────────────────────────────────────────────────` (`src/lib/utils.test.ts`)

**Linting:**
- ESLint flat config: `eslint.config.js`
- Extends: `@eslint/js` recommended, `typescript-eslint` recommended, `eslint-plugin-react-hooks` flat recommended, `eslint-plugin-react-refresh` (vite preset)
- No custom rule overrides beyond the extends list — relies on the plugin defaults
- `dist` is the only global ignore

## Import Organization

**Order (observed, not enforced by lint):**
1. External packages (`react`, `zustand`, `clsx`, `lucide-react`, `@radix-ui/*`, `@tauri-apps/*`)
2. Internal `@/lib/*` and `@/store/*` modules
3. Sibling component imports (relative `./`)
4. Icon/asset imports last when present

Example (`src/components/task/TabBar.tsx:3-19`):
```ts
import { useEffect, useMemo, useRef, useState } from "react";
import type { Task, Tab, TerminalTab, Agent } from "@/lib/types";
import { useApp, useTaskTabs, useActiveTabId } from "@/store/app";
import { getAllLeaves } from "@/lib/splitTree";
import { useTabStripDrag } from "./useTabStripDrag";
import { Button } from "@/components/ui/Button";
...
import { CliIcon, CLI_BRAND_COLOR, CLI_LABEL, resolveIconId } from "@/icons/cli";
import { Plus, X, ... } from "lucide-react";
```

**Path Aliases:**
- `@/*` → `./src/*`, declared in `tsconfig.json` (`paths`) and mirrored in `vite.config.ts` (`resolve.alias`)
- Always use `@/lib/...`, `@/store/...`, `@/components/...` — never deep relative paths (`../../../lib/x`) across top-level directories

## Error Handling

**TypeScript/React:**
- IPC wrapper functions in `src/lib/ipc.ts` return typed Tauri `invoke<T>(...)` promises; callers `await` and rely on the promise rejecting on backend `Result::Err`
- Validation/fallback pattern for persisted/user data: parse defensively, fall back to a safe default rather than throwing — see `parseThemeMode` in `src/store/prefs.ts` (falls back to `"claude"` on unknown/removed theme ids, with an inline comment explaining why)
- No global error boundary framework observed beyond localized try/catch in IPC call sites

**Rust (`src-tauri/`):**
- Uses `anyhow::{anyhow, Context, Result}` as the standard `Result` alias (`src-tauri/src/lib.rs:18`)
- Errors constructed with `anyhow!("message")` and propagated with `?`, e.g. `dirs::home_dir().ok_or_else(|| anyhow!("no home"))?` (`src-tauri/src/lib.rs:558`)
- Tauri commands that need to cross the IPC boundary as `String` errors use `Result<T, String>` explicitly (e.g. `normalize_member`, `src-tauri/src/lib.rs:672`) — `anyhow::Error` does not serialize directly to the frontend, so command-boundary functions convert to `String` while internal helpers use `anyhow::Result`
- Never silently swallow errors: propagate with `?` or `.context("...")`, don't `.ok()`/discard `Result` without a documented reason

## Comments

**When to Comment:**
- File-header comments explain the module's *purpose and non-obvious constraints*, not what the code does line-by-line — see the top of `src/store/prefs.ts`: "User-visible UI preferences (separate from app data and transient UI state). Persisted to localStorage so they survive launches..."
- Comments explain *why*, especially historical/regression context: `// The #15 case: a Linear branch pasted verbatim stays multi-segment.` (`src/lib/utils.test.ts`), `// See issue #32.` (`src/components/task/TabBar.tsx`)
- JSDoc-style `/** ... */` block comments precede exported functions to document behavior/edge cases, e.g. `src/lib/utils.ts` documents `slugify`, `branchify`, `cleanLines`, `shortPath` each with a one-to-few line doc comment
- No multi-paragraph comment blocks explaining obvious code — comments are dense and information-bearing

**JSDoc/TSDoc:**
- Used consistently on exported utility functions in `src/lib/*.ts` (not required on every function, but standard for shared/public helpers)
- Not typically used on internal component props or private helpers

## Function Design

**Size:** Small, focused functions preferred; one-liners common for pure utilities (`src/lib/utils.ts`). Larger files like `GitPanel.tsx` (45.7K) and `MarkdownPreview.tsx` (54.6K) hold many related small functions/handlers rather than one giant function — file size accumulates from breadth of a feature area, not single sprawling functions.

**Parameters:** Options-object pattern for functions with several related parameters, e.g. `decideResume({ isAgent, idCapable, isPrimary, isRepoRoot, hasResumableHistory, storedUuid, resumeOverride, failedResume })` (`src/store/resume.integration.test.ts`) rather than long positional argument lists.

**Return Values:** Prefer returning plain values/typed objects over throwing for expected control flow (e.g. `decideResume` returns a `{ kind: ... }` discriminated union consumed by callers with a switch/ternary on `kind`).

## Module Design

**Exports:** Named exports throughout — no default exports observed for components, hooks, or utilities. Multiple related exports grouped in one file per concern (e.g. `src/lib/customTheme.ts` exports `applyCustomVars`, `clearCustomVars`, `clearThemeCache`, `isCustomId`, `mergeTerminal`, `readThemeCache`, `sanitizeTheme`, `writeThemeCache`, and the `CustomTheme` type together).

**Barrel Files:** Not used — no `index.ts` re-export barrels found under `src/`. Import directly from the defining module (`@/lib/utils`, `@/store/app`), not from a directory barrel.

**Store Pattern (Zustand):** Each store is a single `create<State>()((set, get) => ({...}))` call in its own file under `src/store/`. State mutations go through `set(s => ({ ... }))` (functional updater reading prior state) or `set({ ... })` (direct patch) — never mutate `get()`'s returned objects in place. Cross-cutting reads use `get()` inside actions, not external state capture.

## Rust Conventions (`src-tauri/`)

- Tauri commands are annotated `#[tauri::command]` and predominantly live in `src-tauri/src/lib.rs`; focused concerns are split into separate modules (`shell_env.rs`, `repo_config.rs`, `sandbox.rs`, `proxy.rs`)
- Unit tests are co-located in the same file under `#[cfg(test)] mod tests { use super::*; ... }` at the bottom of the module (see `src-tauri/src/shell_env.rs:340`) — not in a separate `tests/` directory
- Test names are descriptive full sentences in `snake_case`: `fn fallback_path_adds_homebrew_when_missing()`, `fn fallback_path_does_not_duplicate_existing_entry()`
- `assert!(..., "message")` and `assert_eq!(..., "message")` include a failure message explaining the invariant, not just the raw comparison

---

*Convention analysis: 2026-07-12*
