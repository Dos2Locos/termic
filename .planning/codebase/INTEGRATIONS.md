# External Integrations

**Analysis Date:** 2026-07-12

## APIs & External Services

**AI Agent CLIs (primary integration surface):**
- Claude Code CLI (`claude`) - spawned as a child PTY process, not called via API/SDK
  - Invocation: `src-tauri/src/lib.rs` spawns via `portable-pty`, resolves binary from `~/.local/bin` or PATH (see `shell_env.rs`)
  - Session resumption: id-capable, tracked per-tab (`lib.rs` ~line 267/352)
  - Config surfaced via shared `CLAUDE.md` file written into each new task's worktree (`lib.rs` ~line 1783)
- Gemini CLI (`gemini`) - same PTY-spawn pattern, id-capable session resumption; shared `GEMINI.md` written per task
- Codex CLI (`codex`) - same PTY-spawn pattern, but cwd-resume only (no session id), shared `.codex` config dir copied per task
- Default CLI: `"claude"` (`lib.rs` line 1712/1893)
- All three are external binaries the user must have installed; termic does not vendor or call their APIs directly — it drives them as interactive terminal programs over a PTY (`portable-pty`), same as a human would in a shell.

**Update server:**
- `https://termic.dev/updates/latest.json` - Tauri updater endpoint (`tauri-plugin-updater`), configured in `src-tauri/tauri.conf.json` under `plugins.updater.endpoints`
  - Signature verification via embedded minisign pubkey (base64 `pubkey` field in `tauri.conf.json`)
  - Install mode on Windows: `"passive"`

**Git (via shelled-out CLI, not a Rust git library):**
- `std::process::Command::new("git")` invoked directly throughout `src-tauri/src/lib.rs` (worktree creation, branch fetch/checkout, diff, log, status — e.g. lines ~1000, 1048, 4454, 4477, 5639, 5976-6112)
- Used for: creating/removing git worktrees per task, fetching base ref before task creation, diffing, branch listing
- No GitHub/GitLab API calls — all git operations are local CLI invocations against the user's existing repo remotes

**Bash (via shelled-out CLI):**
- `std::process::Command::new("bash")` used for setup/run/archive scripts (`lib.rs` ~lines 3025, 5778, 5816, 6540, 6727) — explicitly NOT sandboxed (only the agent CLI PTY is sandboxed, per `CLAUDE.md`)

## Data Storage

**Databases:**
- None. No SQL/NoSQL database. All persistence is flat files (JSON/YAML) on local disk.

**File Storage:**
- Local filesystem only (per project CLAUDE.md: "app is entirely on-device")
- App data directory (via Rust `dirs` crate) stores: Projects, Tasks, Settings, Tabs — see `docs/data-model.md` for entity layout
- Per-repo config: `.termic.yaml` at target repo root, parsed by `src-tauri/src/repo_config.rs`
- Custom themes: `~/.config/termic/themes/*.json` (see `docs/themes.md`)
- Sound resource bundled: `resources/sounds/choo_choo.caf` (referenced in `tauri.conf.json` bundle resources)

**Caching:**
- None detected (no Redis/memcached/local cache layer)

## Authentication & Identity

**Auth Provider:**
- None built into termic itself. Authentication is entirely delegated to the spawned agent CLIs (`claude`, `gemini`, `codex`), which handle their own auth (API keys / OAuth) independently in their own config dirs. Termic does not intercept, store, or manage credentials for these tools beyond copying their shared config files (`CLAUDE.md`, `.claude/`, `.gemini/`, `.codex/`) into task worktrees.

## Monitoring & Observability

**Error Tracking:**
- None (no Sentry/Bugsnag or similar SDK in dependencies)

**Logs:**
- Custom debug logging macro `dlog` used internally (`src-tauri/src/proxy.rs` imports `crate::dlog`) — local-only, no remote log shipping

## CI/CD & Deployment

**Hosting:**
- Static download/update site at `termic.dev` (not part of this repo)
- App itself is a locally-distributed macOS `.app`/`.dmg`, no server-side deployment for the app runtime

**CI Pipeline:**
- Not detected in this exploration (no `.github/workflows` inspected in this pass — check separately if needed)

## Environment Configuration

**Required env vars:**
- None required at runtime for the app to start; `VITE_MOCK_UPDATE` and `PORT` are optional dev-only overrides
- Per-agent environment variables are user-configurable in Settings (`src/components/settings/AgentsSection.tsx`) and injected into each agent's spawned PTY environment (not global process env)

**Secrets location:**
- No secrets are stored or managed by termic itself. Any API keys used by `claude`/`gemini`/`codex` live in those tools' own config (outside this repo's control). The updater's minisign **public** key is embedded in `tauri.conf.json` (not a secret — used only to verify signed update artifacts).

## Webhooks & Callbacks

**Incoming:**
- None — desktop app, no HTTP server exposed

**Outgoing:**
- None beyond the updater's outbound HTTPS check against `termic.dev`

## Sandbox / Network Policy (adjacent to integrations)

**Outbound network control:**
- `src-tauri/src/proxy.rs` implements an in-process HTTP CONNECT proxy with a regex hostname allowlist, bound to `127.0.0.1:<random port>` per sandboxed task PTY
- Replaces a previously-spawned `tinyproxy` child process; purely custom Rust, platform-neutral
- Supports `CONNECT host:port` (HTTPS tunneling, used by agent CLIs) and plain absolute-form HTTP requests
- Default-deny: anything not matching the allowlist gets HTTP 403
- This is the mechanism by which sandboxed agent-CLI PTYs are allowed to reach specific external hosts (e.g. api.anthropic.com, api.openai.com, generativelanguage.googleapis.com) while everything else is blocked — see `docs/sandbox.md` for the full threat model
- CSP in `tauri.conf.json` additionally restricts the WKWebView itself: `connect-src 'self' ipc: http://ipc.localhost ws: wss: https://termic.dev`, `img-src` allows `https:` broadly (documented accepted exception)

---

*Integration audit: 2026-07-12*
