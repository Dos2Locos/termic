# Technology Stack

**Analysis Date:** 2026-07-12

## Languages

**Primary:**
- TypeScript ~5.7.0 (strict, project references) - all frontend code in `src/`
- Rust (edition 2021) - all backend/native code in `src-tauri/src/`

**Secondary:**
- Shell/Node scripts - `scripts/dev.mjs`, `scripts/dev.signals.test.mjs`, `scripts/changelog.mjs`

## Runtime

**Environment:**
- Node.js v25.9.0 (dev tooling only; no Node runtime ships in the app)
- Tauri 2 desktop shell (Rust binary + WKWebView on macOS)
- No `.nvmrc` present — Node version not pinned in-repo

**Package Manager:**
- npm (root `package.json`, `type: module`)
- Lockfile: `package-lock.json` present
- Cargo for Rust side (`src-tauri/Cargo.toml`, `Cargo.lock` present)

## Frameworks

**Core:**
- React 19.2.6 - UI layer, no SSR, pure SPA rendered into WKWebView
- Vite 8.0.0 - dev server / bundler, `vite.config.ts` (port 1420, HMR on port+1, `@` alias → `./src`)
- Tauri 2 (`@tauri-apps/api` ^2.11.0, Rust crate `tauri = "2"`) - native shell, IPC bridge, window/process management
- Tailwind CSS v4 (`@tailwindcss/vite` ^4.1.16) - styling via `@theme` CSS vars in `src/index.css` (no tailwind.config.js — v4 CSS-first config)
- Zustand 5.0.9 - state management, stores in `src/store/{app,ui,prefs,scriptRuns}.ts`

**Editor / Terminal:**
- CodeMirror 6 (`codemirror`, `@codemirror/*` packages) - code editor, with per-language packages (cpp, css, go, html, java, javascript, json, markdown, python, rust, sql, xml, yaml) plus `@codemirror/legacy-modes`
- `@uiw/codemirror-theme-*` (atomone, aura, copilot, github, gruvbox-dark, nord, tokyo-night, xcode) - editor themes
- xterm.js 5.5.0 (`@xterm/xterm`) + addons: `addon-fit`, `addon-search`, `addon-unicode11`, `addon-web-links`, `addon-webgl`, `addon-clipboard`, `addon-image` - terminal rendering (WebGL-accelerated)

**UI Primitives:**
- Radix UI (`@radix-ui/react-context-menu`, `-dialog`, `-dropdown-menu`, `-hover-card`, `-popover`, `-slot`, `-tabs`, `-tooltip`)
- `lucide-react` - icon set
- `class-variance-authority`, `clsx`, `tailwind-merge` - class composition utilities

**Markdown / Diagrams:**
- `markdown-it` 14.2.0 - markdown rendering (preview pane)
- `dompurify` 3.4.11 - HTML sanitization for rendered markdown
- `mermaid` 11.15.0 - diagram rendering in markdown preview

**Testing:**
- Vitest 3.2.4 - unit test runner (`vitest.config.ts`, `npm test` → `vitest run`)
- `happy-dom` 20.9.0 - DOM environment for Vitest
- No Rust test framework beyond built-in `cargo test` (dev-dependency `tempfile` for fixture dirs)

**Build/Dev:**
- `@vitejs/plugin-react` ^6.0.1
- `@tauri-apps/cli` ^2
- `eslint.config.js` (flat config) - linting
- `scripts/dev.mjs` - custom dev launcher wrapping `tauri dev`

## Key Dependencies

**Critical (Rust, `src-tauri/Cargo.toml`):**
- `portable-pty` 0.8 - PTY spawning for terminal panes (wezterm's crate)
- `tauri-plugin-window-state` 2 - window position/size persistence
- `tauri-plugin-updater` 2.10.1 + `tauri-plugin-process` 2.3.1 - self-update flow
- `tauri-plugin-notification` 2, `tauri-plugin-dialog` 2, `tauri-plugin-opener` 2 - OS integration
- `serde` / `serde_json` 1 - IPC payload (de)serialization
- `serde_yml` 0.0.12 - parses repo-root `.termic.yaml` config (maintained fork of serde_yaml)
- `regex` 1.12.3 - proxy hostname allowlist, file-tree exclude matching
- `glob` 0.3 - file-tree exclude pattern matching
- `chrono` 0.4 (serde feature) - timestamps
- `uuid` 1 (v4, serde) - task/session ids
- `parking_lot` 0.12 - locking primitives
- `anyhow` 1 - error handling
- `dirs` 5 - platform data dir resolution
- `font-kit` 0.14 - enumerates installed monospace fonts (lazy-evaluated, only on-demand)
- `base64` 0.22 - data: URL encoding for workspace images in markdown preview
- `percent-encoding` 2 - decodes `taskpdf:` URI scheme (native PDF preview)
- `libc` 0.2 - low-level process signal handling (SIGKILL on sandbox teardown)
- macOS-only: `objc2` 0.6, `objc2-foundation` 0.3 (`NSProcessInfo` feature) - raw Objective-C bridge to round window corners on macOS Tahoe (26+)

**No HTTP client crate** (`reqwest`/`ureq`/`hyper` absent) — outbound HTTP is handled by the custom in-process CONNECT proxy in `src-tauri/src/proxy.rs`, not by the app itself calling out.

**Infrastructure:**
- `@tauri-apps/plugin-updater`, `-process`, `-dialog`, `-notification`, `-opener` (frontend bindings matching the Rust plugins above)

## Configuration

**Environment:**
- No `.env` file support in the app itself; `import.meta.env.DEV` (Vite-injected) gates dev-only UI (e.g. `src/main.tsx`, `src/components/UpdaterBanner.tsx`)
- `VITE_MOCK_UPDATE=available|whatsnew` env var mocks the update-available UI in dev (`npm run tauri:dev`)
- `PORT` env var overrides Vite dev server port (default 1420); read by both `vite.config.ts` and `scripts/dev.mjs`
- Per-repo project config: `.termic.yaml` at target repo root (parsed via `serde_yml` in `src-tauri/src/repo_config.rs`)
- Agent CLI env vars configurable per-agent in Settings (`src/components/settings/AgentsSection.tsx`), stored in app settings, injected into spawned PTYs

**Build:**
- `vite.config.ts` - frontend bundling (React + Tailwind v4 plugins, `@` path alias)
- `tsconfig.json` + `tsconfig.app.json` + `tsconfig.node.json` - TS project references, `@/*` → `./src/*`
- `eslint.config.js` - flat ESLint config
- `vitest.config.ts` - test runner config
- `src-tauri/tauri.conf.json` - Tauri app config: product name "Termic", identifier `com.simion.termic`, CSP policy, bundle targets "all", macOS min system version 12.0, entitlements/Info.plist references
- `src-tauri/capabilities/default.json` - Tauri permission capability set for the main window (core window controls, opener, notification, dialog, window-state, updater, process — all scoped to `windows: ["main"]`)
- `src-tauri/Entitlements.plist`, `src-tauri/Info.plist` - macOS bundling metadata

## Platform Requirements

**Development:**
- Node.js (v25.9.0 observed; no enforced minimum pinned)
- Rust toolchain + Cargo (edition 2021)
- macOS primary target (WKWebView-based); Tauri also supports other OSes but this project's docs/CLAUDE.md focus exclusively on macOS behavior (e.g. Tahoe corner-rounding, sandbox-exec)

**Production:**
- macOS 12.0+ (`minimumSystemVersion` in `tauri.conf.json`)
- Distributed as `.app`/`.dmg` via `npm run tauri:build` → `src-tauri/target/release/bundle/`
- Self-updates via `tauri-plugin-updater` against `https://termic.dev/updates/latest.json`, signature-verified with an embedded minisign pubkey

---

*Stack analysis: 2026-07-12*
