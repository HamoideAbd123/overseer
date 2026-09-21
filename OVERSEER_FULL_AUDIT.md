# Overseer full repository audit

Audit date: 2026-09-21. Scope: read-only inspection of the repository as present. No existing project file was changed. The only artifact created by this audit is this report.

## Executive architecture

Overseer is a self-hosted, single-Go-binary fleet-control product. `overseer serve` hosts the embedded authenticated React control plane, a REST API, WebSocket bridges, an HTTP MCP endpoint, and SQLite storage. It starts a local in-process agent for the hub machine. Remote devices run the same binary as an outbound-only WebSocket agent; the hub relays terminal streams, control messages, commands, files, and statistics. The binary also contains a token-authenticated fleet CLI and a stdio MCP server.

The repository also contains a separate Vite marketing site in `site/`. It is not embedded in, nor required by, the hub binary. `internal/salmon/` is present but untracked and isolated from all production call paths.

## Repository state and structure

`git status --short` before this report showed pre-existing user/worktree changes: `D cmd/overseer/uidist/.gitkeep`, `M ui/tsconfig.tsbuildinfo`, and `?? internal/salmon/`. The source inventory below includes Salmon because it exists in the working tree and was specifically requested; it is not tracked by Git. Ignored generated output currently exists beneath `cmd/overseer/uidist/assets/`; it is not source and is omitted from the file-by-file implementation list.

| Path | Purpose |
|---|---|
| `.claude/` | Local launcher configuration for the hub and the marketing site. |
| `.github/workflows/` | CI, cross-platform build, installer regression, release, checksum, and provenance automation. |
| `cmd/overseer/` | Go executable entry point and `go:embed` UI boundary. `uidist/` is generated staging content. |
| `docs/superpowers/specs/` | Approved product/design specification. |
| `internal/agent/` | Device-side connectivity, command execution, terminals, sessions, file transfer, service installation, and agent self-update. |
| `internal/fleet/` | Authenticated REST client shared by the fleet CLI and stdio MCP server. |
| `internal/hub/` | HTTP/TLS server, API, auth, SQLite store, live-agent registry, browser WebSockets, installers, setup, projects, MCP HTTP, and hub updates. |
| `internal/mcp/` | JSON-RPC/MCP protocol handling and tool implementations. |
| `internal/protocol/` | Shared hub-agent control messages and binary-frame format. |
| `internal/salmon/` | Experimental Gemini-backed `salmon.Agent`; currently disconnected/untracked. |
| `internal/updater/` | Shared GitHub release discovery, checksum validation, binary replacement, and rollback. |
| `scripts/` | Linux/macOS/Windows hub installers and deterministic installer test harnesses. |
| `ui/` | Authenticated embedded React 18 TypeScript control-plane application. |
| `site/` | Separate public React 19 marketing/waitlist site; mirrored public installer downloads and optional screenshots. |
| Root files | `go.mod`/`go.sum` pin Go dependencies; `Makefile` builds/tests/releases; `README.md` is operating documentation; `LICENSE` is MIT; `.gitignore` ignores outputs/data. |

## Source-file inventory

### Root, command, documentation, and automation

| File | Language | Responsibility and connections |
|---|---|---|
| `Makefile` | Make | `all` builds UI then Go; `ui` runs npm build and stages `ui/dist` in `cmd/overseer/uidist`; `go`, `test`, `cross`, and destructive-to-build-output-only `clean` targets. Cross-builds six OS/architecture targets with `CGO_ENABLED=0`. |
| `README.md` | Markdown | Product, installation, TLS/Tailscale, fleet CLI, local/remote MCP, update, and developer documentation. It accurately describes most implemented flows, but should not be treated as an authority over code. |
| `go.mod` | Go module | Module `github.com/ErzenXz/overseer`, Go 1.25. Direct dependencies: ConPTY, Unix PTY, Gorilla WebSocket, gopsutil, x/crypto, x/sys, and pure-Go SQLite. |
| `go.sum` | Go checksum data | Integrity lock data for direct and transitive Go modules. |
| `.gitignore` | Git config | Ignores binaries, dist/UI outputs, local SQLite data, and common editor/log files; preserves only the UI embed placeholder. |
| `LICENSE` | Text | MIT license, copyright 2026 Erzen Krasniqi. |
| `.claude/launch.json` | JSON | Launches `./overseer serve` with `/tmp/overseer-dev-data`, and Vite marketing site on port 5180. |
| `docs/superpowers/specs/2026-07-03-overseer-design.md` | Markdown | Approved original design. Useful design intent; partially stale: it lists Windows as non-goal and an older API shape, whereas implementation supports Windows agents, projects/fx, updates, and remote MCP. |
| `.github/workflows/ci.yml` | GitHub Actions YAML | Runs Go vet/tests (installs tmux), builds/cross-compiles after UI build, and runs Unix/Windows installer fallback tests. It does not build/test the `site/` app. |
| `.github/workflows/release.yml` | GitHub Actions YAML | On `v*` tag builds six raw binaries, archives them, creates `checksums.txt`, submits GitHub artifact attestations, and publishes release assets. |
| `cmd/overseer/main.go` | Go | Main command dispatcher. Implements `serve`, `agent enroll/run/install-service`, `fleet login/devices/sessions/new/run/send/read/kill`, `mcp`, `update`, `rollback`, version/help. Injected `version` and `githubRepo` are release-time linker variables. Constructs `hub.Options`, `agent`, `fleet.Client`, `mcp`, and `updater` calls. |
| `cmd/overseer/ui.go` | Go | Embeds generated `uidist` with `go:embed`; `uiFS()` returns it only if `index.html` exists. This cleanly isolates serving from frontend build staging. |
| `cmd/overseer/uidist/.gitkeep` | Placeholder | Tracked empty-file placeholder required by `go:embed` before a UI build. It is currently deleted in the pre-existing dirty worktree. |

### Shared protocol and updater

| File | Language | Responsibility and important types/functions |
|---|---|---|
| `internal/protocol/protocol.go` | Go | Hub-agent wire contract. `Msg` is JSON control envelope; `Hello`, `Welcome`, `Stats`, `Session*`, `Exec*`, `Term*`, and `Fs*` payloads model requests. Text WebSocket frames are JSON; `EncodeFrame`/`DecodeFrame` carry binary stream data prefixed by a big-endian `uint32` channel ID. Used by `internal/hub` and `internal/agent`. |
| `internal/protocol/protocol_test.go` | Go test | Covers frame round trip, empty payload, short-frame rejection, and JSON message/payload round trip. |
| `internal/updater/updater.go` | Go | Shared release/update implementation. `Latest`/`ForVersion` query GitHub REST; stable SemVer parsing gates releases; `fetchChecksum`, `downloadStaged`, and `validateBinary` verify SHA-256 and the staged binary's `version` output. `Install`, `Rollback`, `swapUnix`, and `scheduleWindowsSwap` retain one previous binary and atomically/handoff replace it. Used by hub, device agent, and CLI. |
| `internal/updater/updater_test.go` | Go test | Tests version ordering and mocked GitHub release asset/checksum selection/rejection. It does not execute actual download/swap/rollback paths. |

### Hub backend

| File | Language | Responsibility and important types/functions |
|---|---|---|
| `internal/hub/server.go` | Go | Core `Server` and `Options`; selects `~/.overseer`, opens store, creates auth/event/agent registry/update manager, sets routes, runs HTTP or autocert TLS, starts a loopback internal listener and embedded hub agent. Implements UI SPA fallback and JSON helpers. |
| `internal/hub/auth.go` | Go | Password Argon2id hashing/constant-time verification; 30-day in-memory browser `sessionManager`; per-IP fixed-window login limiter (10/minute); `requireAuth` accepts cookie or bearer API token. Browser sessions vanish on restart. |
| `internal/hub/store.go` | Go | SQLite schema and data layer. Tables: `settings`, `devices`, `enroll_tokens`, `presets`, `api_tokens`, `projects`; WAL plus one open connection. Stores hashes, not plaintext, of device/enrollment/API tokens. Main models: `Device`, `Project`, `Preset`, `ApiToken`; seeds Shell/Claude/Codex presets. |
| `internal/hub/api.go` | Go | REST handlers: first setup/login/logout/me; enrollment; device list/rename/delete; updates; sessions; session input/output; arbitrary exec; fleet session aggregation; presets/tokens; file list/download/upload. It maps browser/API authority to live `agentConn` protocol calls and enforces timeouts. |
| `internal/hub/conns.go` | Go | `agentConn` serializes WebSocket writes, correlates request IDs, allocates stream channels, tracks stats; `registry` owns current device connections; `eventBus` broadcasts online/offline/stats/session changes. `serveAgent` is the hub read loop. |
| `internal/hub/ws.go` | Go | Gorilla WebSocket upgrader, same-origin `checkOrigin`, device-agent handshake (`handleAgentWS`), browser terminal bridge (`handleTermWS`), and browser event feed (`handleEventsWS`). |
| `internal/hub/install.go` | Go | Generates POSIX/PowerShell one-paste agent installers, strictly token-sanitizes interpolated enrollment token, and serves platform binary locally or redirects to GitHub release. |
| `internal/hub/projects.go` | Go | Project CRUD validation and `handleProjectExec`, the constrained fx host boundary: project selects device/cwd server-side while browser supplies command/timeout. |
| `internal/hub/setup.go` | Go | Fixed `setupSpecs` catalogue for Node, Codex, Claude Code, Gemini CLI, and Tailscale. Probes live devices and returns fixed installation/auth commands; UI cannot submit arbitrary setup commands through this route. |
| `internal/hub/update.go` | Go | Hub `updateManager`: persisted auto-update preference, managed-service gating, startup + six-hour check schedule, release status, async install/rollback callbacks. |
| `internal/hub/mcp_http.go` | Go | `/mcp` Streamable-HTTP-style POST JSON-RPC gateway. Verifies bearer API token, limits body to 8 MiB, then invokes shared MCP code through the loopback API client. |
| `internal/hub/store_test.go` | Go test | Device/enrollment/token/password/preset/project lifecycles and project cascade on device deletion. |
| `internal/hub/integration_test.go` | Go test | Starts live hub + in-process enrolled agent over loopback; tests exec, non-zero exit, filesystem listing, tmux session create/list/kill, bad session rejection, browser-password flow, and API token auth. tmux test is conditional. |
| `internal/hub/term_test.go` | Go test | Full browser-WebSocket → hub → agent PTY → tmux terminal bridge; skipped if tmux unavailable. |
| `internal/hub/install_test.go` | Go test | Redirect behavior, path traversal rejection, installer token sanitization, Unix/Windows script content, enrollment command generation. |
| `internal/hub/mcp_http_test.go` | Go test | `/mcp` missing/bad token rejection, initialization, file tool declaration, command tool execution, notification 202 response. |
| `internal/hub/setup_test.go` | Go test | Fixed cross-platform setup catalogue and remote-friendly login command assertions. |
| `internal/hub/update_test.go` | Go test | Persists auto-update setting and validates non-managed status. |

### Device agent

| File | Language | Responsibility and important types/functions |
|---|---|---|
| `internal/agent/agent.go` | Go | `Config`, `Agent`, managed PATH augmentation, outbound authenticated agent WebSocket reconnect loop (jittered exponential backoff to 30s), hello/welcome, dispatch, serialized writes, channel maps, and teardown. It never listens on a device port. |
| `internal/agent/enroll.go` | Go | Exchanges one-time enrollment token for permanent device token; writes `~/.overseer/agent.json` with directory mode 0700/file mode 0600. |
| `internal/agent/ops.go` | Go | Five-second gopsutil stats; `execCommand` uses caller user's shell/PowerShell with 60s default/10m max and 256 KiB each stdout/stderr cap; directory listing; regular-file download; streaming upload to temporary file then rename. |
| `internal/agent/sessions.go` | Go | Validates session names, tmux-backed create/list/kill with `@overseer_kind`, activity heuristic, and global process-local ephemeral PTY fallback when tmux is absent. |
| `internal/agent/terminal.go` | Go | `termStream`, terminal dimension clamping, session attach/new tmux operation, binary PTY pump, resize, detach. |
| `internal/agent/terminal_unix.go` | Go | Non-Windows build: `creack/pty` implementation and process cleanup. |
| `internal/agent/terminal_windows.go` | Go | Windows build: `charmbracelet/x/conpty` implementation, spawned handle tracking, resize, and process termination/wait fallback. |
| `internal/agent/service.go` | Go | Installs a managed agent under root/system or user systemd, macOS LaunchAgent, or Windows ONLOGON scheduled task. Writes service definitions and invokes service managers. |
| `internal/agent/update.go` | Go | Agent update gating: only managed services, only newer stable hub-advertised release and configured repo; retries hourly after failed attempt/15-minute healthy-connection loop. |
| `internal/agent/path_test.go` | Go test | Ensures managed PATH additions never become relative paths with missing home data. |
| `internal/agent/update_test.go` | Go test | Tests self-update gating, not an actual update. |

### Fleet CLI and MCP

| File | Language | Responsibility and important types/functions |
|---|---|---|
| `internal/fleet/client.go` | Go | Token-authenticated 120-second REST client; `SaveConfig` stores `~/.overseer/fleet.json` mode 0600; uses `OVERSEER_HUB`/`OVERSEER_TOKEN` or file. Provides devices/session/exec calls for CLI and MCP. |
| `internal/mcp/mcp.go` | Go | Minimal MCP JSON-RPC server for newline-delimited stdio and reusable HTTP handling. Supports `initialize`, `tools/list`, `tools/call`, `ping`. Tools: `list_devices`, `list_sessions`, `create_session`, `send_input`, `read_output`, `run_command`, `kill_session`, `list_files`, `read_file`, `write_file`. File tools intentionally wrap shell utilities; write uses base64/heredoc and caps content at 512 KiB. |

### Salmon

| File | Language | Current responsibility and integration status |
|---|---|---|
| `internal/salmon/salmon.go` | Go | Defines `Agent` with Gemini/Overseer fields, `New`, public `ChatRequest`/`ChatResponse`, Gemini request/response DTOs, and `Chat`. `New` reads `GEMINI_API_KEY` and `OVERSEER_TOKEN`, hard-codes model `gemini-2.5-flash` and hub URL `http://127.0.0.1:4200`. `Chat` calls Gemini `generateContent` with a prompt that says fleet execution will be connected “in the next step.” It never uses `OverseerURL`/`OverseerToken`, `internal/fleet`, or `internal/mcp`; no package imports it and no route/CLI registers it. It is isolated and incomplete. Formatting is non-idiomatic and it has no tests. |

### Authenticated frontend (`ui/`)

`ui/` is React 18 + TypeScript strict mode + Vite 6 + Tailwind 4. `main.tsx` uses `BrowserRouter`; `App.tsx` gates routes by `/api/me` and browser cookie state. State is localized React state/hooks—there is no Redux/query library/global store. Vite proxies `/api` (including WebSockets) and `/install` to `localhost:4200` in development. Production output is embedded through `cmd/overseer/ui.go`.

| File | Responsibility |
|---|---|
| `ui/package.json`, `ui/package-lock.json` | npm package/lock: React, React Router, xterm + fit addon, libfx, Vite, Tailwind, TypeScript. Scripts `dev`, `build`, `preview`. |
| `ui/vite.config.ts` | React/Tailwind plugins and Go-hub development proxy. |
| `ui/tsconfig.json` | ES2022 strict no-emit TypeScript configuration. `ui/tsconfig.tsbuildinfo` is generated build metadata and currently modified pre-audit. |
| `ui/index.html` | Root shell and inline favicon. |
| `ui/src/main.tsx` | React root, strict mode, browser router, global CSS. |
| `ui/src/App.tsx` | Authentication bootstrap/unauthorized redirect and routes: Code, Fleet, device, agents, setup, settings, login. |
| `ui/src/api.ts` | Same-origin JSON API wrapper, `ApiError`, unauthorized callback, project exec fetch, WebSocket URL builder, display helpers. Cookies are implicit `fetch` same-origin credentials. |
| `ui/src/hooks.ts` | `useHubEvents` reconnecting event WebSocket and `usePoll`. |
| `ui/src/types.ts` | API-facing TypeScript interfaces for stats, devices, projects, sessions, setup, tokens, updates, filesystem, and events. |
| `ui/src/index.css` | Tailwind import plus global dark design tokens, reusable component classes, typography, responsive/terminal styling. |
| `ui/src/components/Layout.tsx` | Persistent nav/shell, mobile navigation, product branding/icons, `Outlet`. |
| `ui/src/components/Modal.tsx` | Reusable accessible-ish overlay/card shell with close callback. |
| `ui/src/components/StatusBadge.tsx` | Device/session status label presentation. |
| `ui/src/components/AddDeviceModal.tsx` | Mints enrollment token, presents Unix/Windows command, clipboard copy, reacts to device-online event. |
| `ui/src/components/LaunchSessionModal.tsx` | Lists presets and posts session name/cwd/command/kind to selected device. |
| `ui/src/components/Terminal.tsx` | xterm terminal client to authenticated `/api/ws/term`; fits/resizes, forwards keystrokes and renders binary output. |
| `ui/src/components/FileBrowser.tsx` | Device list/navigate/download/upload UI using filesystem API; uploads via `FormData`. |
| `ui/src/components/ProjectModal.tsx` | Project add/edit form selecting device and path. |
| `ui/src/components/FxTerminal.tsx` | Integrates browser `libfx` terminal runtime with project-restricted `/api/projects/{id}/exec`; includes UTF-8 payload limit and notices. |
| `ui/src/lib/fxStorage.ts` | Browser IndexedDB adapter for fx credentials/settings/session/prompt history. This is browser-local, not hub database state. |
| `ui/src/lib/libfx.d.ts` | Local declaration for the `libfx/browser` API. |
| `ui/src/pages/Login.tsx` | First-run setup/password login form and API error display. |
| `ui/src/pages/Dashboard.tsx` | Fleet overview, polling/event-driven device cards and health meters; opens add-device flow. |
| `ui/src/pages/DevicePage.tsx` | Per-device details, polling sessions, terminal/files tabs, launch and device management. |
| `ui/src/pages/AgentsPage.tsx` | Fleet-wide session/agent visibility using `/api/agents`. |
| `ui/src/pages/CodingPage.tsx` | Project selection/CRUD, device association, and `FxTerminal` workspace. |
| `ui/src/pages/SetupPage.tsx` | Chooses online device, requests fixed setup catalogue, launches setup/login sessions. |
| `ui/src/pages/SettingsPage.tsx` | Update controls/status, API-token one-time reveal/revoke, preset CRUD, local and remote MCP instructions. |

### Marketing site (`site/`)

`site/` is a distinct static React 19/Vite 8/Tailwind 4 application. It is a product/installer landing page, never embedded in the binary. It contains no call to a hub API. `site/public/shots/.gitkeep` reserves optional screenshots; source renders placeholders if images are missing.

| File/group | Responsibility |
|---|---|
| `site/package.json`, `site/package-lock.json` | Site dependencies: React 19, Phosphor icons, Geist fonts, Vite 8, Tailwind, Oxc lint. Scripts `dev`, `build`, `lint`, `preview`. |
| `site/vite.config.ts`, `site/tsconfig*.json`, `site/.oxlintrc.json`, `site/.gitignore` | Vite, TS project references, hook/type lint rule set, and site-specific output/env ignores. |
| `site/index.html` | SEO/OpenGraph page shell and motion opt-in respecting reduced-motion preference. |
| `site/src/main.tsx`, `site/src/App.tsx`, `site/src/index.css` | React root, composed single-page section order, global dark visual system. |
| `site/src/lib/site.ts` | GitHub/release/docs constants and optional `VITE_WAITLIST_ENDPOINT`. |
| `site/src/lib/reveal.tsx` | CSS-based `Reveal` IntersectionObserver wrapper; content stays visible without JS/reduced motion. |
| `site/src/components/Nav.tsx` | Responsive navigation and GitHub/install links. |
| `site/src/components/Hero.tsx` | Product hero and CTAs. |
| `site/src/components/Install.tsx` | Platform selector/installer command presentation. |
| `site/src/components/Capabilities.tsx` | Capability grid and screenshot references. |
| `site/src/components/FleetAgents.tsx` | Fleet/agent feature presentation. |
| `site/src/components/HowItWorks.tsx` | Hub/spoke explanatory diagram/content. |
| `site/src/components/Exposure.tsx` | Network exposure/TLS/Tailscale messaging. |
| `site/src/components/OverseerCode.tsx` | Overseer Code section and optional endpoint-backed waitlist form. |
| `site/src/components/Screenshot.tsx` | Image-or-labelled-placeholder component for optional screenshots. |
| `site/src/components/Footer.tsx` | Footer links. |
| `site/README.md` | Site development, screenshot, and waitlist documentation. |
| `site/public/favicon.svg` | Site favicon. |
| `site/public/install.sh`, `site/public/install-macos.sh`, `site/public/install.ps1` | Exact duplicates of counterpart root installer scripts at audit time (`cmp` returned identical). Duplication creates a future drift risk. |

### Installer scripts

| File | Language | Responsibility |
|---|---|---|
| `scripts/install.sh` | POSIX shell | Linux hub installer/uninstaller: packages, release download/checksum, source fallback with temporary Go/Node, installation, systemd unit, optional TLS configuration, guarded purge behavior. |
| `scripts/install-macos.sh` | POSIX shell | macOS hub LaunchAgent install/uninstall; release verification/source fallback and plist generation. |
| `scripts/install.ps1` | PowerShell | Windows hub scheduled-task installer/uninstaller; checksum, source fallback, temporary toolchains, task lifecycle. |
| `scripts/test-installers.sh` | POSIX shell | Deterministic Linux/macOS installer regression harness using command stubs and no-release fallback simulations. |
| `scripts/test-installers.ps1` | PowerShell | Windows installer regression harness using mocked web/task/tool commands. |

## Backend and API

### Startup, configuration, storage, logging

The entry point is `cmd/overseer/main.go`. `serve` accepts `--addr` (default `:4200`), `--data-dir` (default `~/.overseer`), and `--tls-domain` plus mandatory `--tls-email`. `internal/hub/server.go` creates data dir mode 0700, opens `<data-dir>/overseer.db`, registers HTTP routes, starts an internal loopback listener, and starts the embedded local agent.

Plain mode listens on the supplied address (therefore all interfaces by default). TLS mode uses `golang.org/x/crypto/acme/autocert`, port 80 for challenge/redirect and port 443 for TLS; certificate cache is `<data-dir>/certs`. The server has a 10-second header-read and 120-second idle timeout; WebSockets manage their own lifecycle. The code uses standard `log.Printf` primarily for listener/agent/update operational events, returns structured `{ "error": ... }` API errors, and bounds request/device waits. There is no structured logging, log level/configuration, request ID, audit log, metrics endpoint, or database migration framework.

SQLite uses WAL and a one-connection limit. Persistent records are settings (including password hash and hub device token), devices, enrollment-token hashes, presets, API-token hashes, and project metadata. Sessions remain a tmux/agent runtime concern, not a database table. Browser sessions are memory-only.

### Route map

Public routes are intentionally limited, but “public” here means unauthenticated—not necessarily safe to expose over plain HTTP:

| Method/path | Handler | Purpose/auth |
|---|---|---|
| `POST /api/setup` | `handleSetup` | First writer creates administrator password; then gets session cookie. |
| `POST /api/login`, `POST /api/logout`, `GET /api/me` | auth handlers | Password/cookie lifecycle and UI status. |
| `POST /api/enroll` | `handleEnroll` | Burns one-time enrollment token and returns permanent device credential. |
| `GET /install/{token}.sh|ps1` | `handleInstallScript` | Serves per-token joining installer. |
| `GET /api/agent-binary` | `handleAgentBinary` | Serves matching local binary or redirects to release asset. |
| `GET /api/ws/agent` | `handleAgentWS` | Permanent device bearer token; receives agent websocket. |
| `POST /mcp` | `handleMCPHTTP` | Bearer API token only; remote MCP JSON-RPC. `GET` returns 405. |

All routes below use `requireAuth`: a valid HttpOnly browser cookie or a bearer API token. Paths are registered in `internal/hub/server.go`.

| Route family | Operations |
|---|---|
| `/api/devices` | List, rename, delete; per-device session list/create/kill/input/output; synchronous exec; setup probe; filesystem list/download/upload. |
| `/api/enroll-tokens` | Create 15-minute, single-use enrollment token and installer commands. |
| `/api/projects` | List/create/update/delete projects and project-restricted exec. |
| `/api/agents` | Aggregate live sessions across agents. |
| `/api/presets`, `/api/tokens` | Preset CRUD and API-token list/create/revoke. |
| `/api/updates` | Status, release check, auto-update setting, async install/rollback. |
| `/api/ws/term`, `/api/ws/events` | Authenticated browser terminal proxy and event stream. |

## Agent and device-control system

1. An authenticated user creates a one-time enrollment token. The generated installer downloads the appropriate binary, invokes `overseer agent enroll`, then installs an OS service/task.
2. Enrollment uses `POST /api/enroll`; a 32-byte random token is atomically marked used and exchanged for a random permanent device token. The device config persists its hub URL/device ID/token locally.
3. `agent.Run` converts the hub scheme to `ws`/`wss`, sends its token in `Authorization: Bearer`, sends `hello`, receives `welcome`, then reconnects forever with jittered exponential backoff. Device-to-hub communication is outbound only.
4. The hub authenticates the device token against its SHA-256 hash, keeps one live connection per device, and supersedes old connections. The hub sends requests containing IDs; agent replies with `result`. Stream channels carry terminal/file traffic in binary frames.
5. The agent samples CPU/memory/swap/disk/network/process/uptime every five seconds via gopsutil. The hub holds only current stats in memory and pushes events to browsers.
6. A session is a validated-name tmux session when tmux is available. The browser terminal opens a hub channel; hub sends `term.open`; agent runs a PTY attached to `tmux new-session -A`; raw terminal bytes are relayed. Without tmux, a process-local ephemeral session exists only while attached.
7. Synchronous `exec` runs through the managed user’s `$SHELL -c` (or `powershell.exe -NoProfile -Command`) subject to 60-second default/600-second maximum and 256 KiB output caps. File browsing uses direct filesystem APIs; transfers use stream channels and upload temp-then-rename.

The security boundary is authorization, not sandboxing: any browser/API/MCP caller with hub authority can execute as the OS user operating the agent and can browse/read/upload arbitrary accessible paths. The agent has no allowlist, root restriction, per-device permission, project directory restriction for general API/MCP, or command policy.

## MCP implementation and AI-agent interaction

There are two transports over the same `internal/mcp/mcp.go` JSON-RPC handler:

- Local stdio: `overseer mcp`, normally configured after `overseer fleet login` or via `OVERSEER_HUB`/`OVERSEER_TOKEN`. It reads newline-delimited JSON and writes newline-delimited JSON responses.
- Remote HTTP: `POST /mcp` with `Authorization: Bearer ovsr_...`, body up to 8 MiB. The hub verifies the token, creates an internal-loopback `fleet.Client` using that same token, and calls `mcp.HandleMessage`. It supports MCP initialize/tools list/tools call/ping and notification 202 semantics, but no SSE/server-initiated stream.

Tools are unrestricted fleet operations: inventory/session management, interactive terminal steering, command execution, and read/write files. The tool implementation resolves devices by ID/name through the standard API. `list_files`, `read_file`, and `write_file` currently shell out to `ls`, `cat`, and `base64 -d`; this is Unix-oriented and conflicts with advertised Windows device support.

An external AI client must reach the hub over a securely exposed HTTPS endpoint, send a valid API token as Bearer credential, call `initialize`/`tools/list`, and then call a tool. The code does not distinguish a remote MCP client from a local CLI/API token: all tokens have identical, fleet-wide, read/write/command authority.

## Frontend communication and state

The browser communicates same-origin using cookie-based `fetch`, with automatic cookie inclusion. `api.ts` sends JSON; `/api/projects/{id}/exec` is special-cased for libfx. API 401s call the App-level redirect handler. Terminal and events use URLs created from `location.protocol`/`location.host` so HTTP→WS and HTTPS→WSS. Pages use local `useState`/`useCallback` with polling; `useHubEvents` provides only event-driven refresh signals. fx stores its browser runtime state and prompt history through IndexedDB adapter code in `ui/src/lib/fxStorage.ts`.

The principal UI routes are `/code` and `/code/:projectId`, `/fleet`, `/devices/:id`, `/agents`, `/setup`, and `/settings`; unauthenticated routes render `/login`. The dashboard default redirects to Code. Styling is Tailwind utility use plus the global CSS design system—no CSS-module/component-scoped stylesheet architecture.

## Build, release, deployment, and development

- Go: Go 1.25 module; pure-Go SQLite enables CGO-free cross builds. `make` runs `cd ui && npm install --no-audit --no-fund && npm run build`, stages UI, and `go build`s `./overseer`. This means building mutates generated output/node dependencies by design.
- Tests: `make test` runs `go vet ./...` then `go test ./...`. `make cross` builds UI then darwin/linux/windows x amd64/arm64 to `dist/`.
- Local hub: `go run ./cmd/overseer serve`; local hot UI: `cd ui && npm install && npm run dev` (Vite proxy to port 4200). Marketing site: `cd site && npm install && npm run dev`; separate `npm run build` and `npm run lint`.
- Deployment: root installers implement Linux systemd, macOS LaunchAgent, Windows scheduled task. Hub has built-in ACME HTTPS, but default remains public-interface plain HTTP. A reverse proxy or Tailscale Serve is supported operationally, not embedded tunneling.
- Release: GitHub tag pipeline publishes raw per-platform binaries + archives + SHA-256 manifest and GitHub provenance attestation. Code verifies checksums and `version` but does not verify attestation/provenance.

## Test audit

There are Go tests only; no React/UI unit tests, browser E2E tests, site tests, fuzz tests, static security scanners, database concurrency tests, or MCP conformance suite. CI does not run site lint/build.

Covered areas are listed in the file inventory: protocol framing; path/update gates; store/auth/token/project lifecycle; installer sanitization; setup catalogue; update preference; hub/agent integration (including exec/files/tmux); terminal WebSocket bridge; and basic remote MCP endpoint behavior.

Important missing coverage:

- Authentication edge cases: TLS proxy cookie behavior, setup bootstrap race, CSRF-like state-changing browser requests, session expiry/restart behavior, login limiter concurrency/IP proxy handling.
- Authorization/scopes/roles (none exist), device-token rotation, compromised/deleted device behavior, audit logging.
- Windows execution, terminal, installer, and MCP file tools on actual Windows; macOS service behavior; non-tmux session REST methods.
- Large/malicious multipart uploads, stream disconnect/error races, download limits, filesystem path/symlink policy, abnormal WebSocket frames/concurrency.
- Full updater download/checksum/swap/rollback on each platform and release provenance verification.
- All UI flows, project/fx behavior, accessibility, route error states, and marketing-site build/lint/form behavior.

## Security audit

### Controls present

- Administrator password is Argon2id (`3` iterations, 64 MiB, parallelism 2), salt 16 bytes; comparison is constant-time.
- Browser session tokens and stored API/device/enrollment tokens are random 32-byte values. Cookies are HttpOnly, SameSite=Lax, 30 days; browser sessions are volatile. API/device/enrollment tokens are stored as SHA-256 hashes and API tokens can be revoked.
- Enrollment tokens are single-use and expire in 15 minutes. Scripts strictly constrain embedded token syntax. OS/arch binary request values are constrained before filesystem/redirect use.
- Device agents authenticate their outbound WebSocket before protocol handling. Browser terminal/event WebSockets require normal auth and enforce same-origin when an Origin header exists.
- API/MCP body sizes and command/terminal/output timeouts are bounded in several important paths; upload commits through a temporary file then rename. Downloads reject non-regular files.
- TLS can be automatically provisioned via ACME; installer/update releases use HTTPS and the updater verifies checksum and reported binary version.

### Confirmed issues / implementation facts with security impact

1. **Critical design exposure: all API tokens are fleet-wide root-equivalent capabilities.** `internal/hub/auth.go` and `internal/hub/mcp_http.go` grant any valid API token every protected endpoint; `internal/mcp/mcp.go` exposes arbitrary shell command and file write. There are no scopes, roles, per-device restrictions, command policies, or token expiry. This is documented as a remote shell, but remains the dominant security risk.
2. **High: unauthenticated first-run setup is a bootstrap takeover window.** `POST /api/setup` accepts the first password while no password hash exists. Because default binding is `:4200`/all interfaces, anyone able to reach an uninitialized hub can claim administrator control. This is confirmed behavior, not merely a recommendation.
3. **High when HTTP is exposed: hub traffic includes passwords, browser session cookies, enrollment tokens, device tokens, remote commands, terminal data, and file data in cleartext.** Plain HTTP/WS is the default in `server.go`; device agent scheme conversion preserves it. README warns users, but the code does not require/redirect to TLS or restrict non-loopback plain HTTP.
4. **High: default command/file capabilities are intentionally unrestricted OS-user authority.** `agent.execCommand`, filesystem functions, terminal PTYs, and MCP tools impose no sandbox or path allowlist. Compromise of a browser session/token/device control-plane channel means access equivalent to the agent’s operating user, including secrets readable by that user.
5. **Medium: TLS-terminating reverse proxies do not produce Secure cookies unless they make the backend request TLS.** `issueSession` sets `Secure: r.TLS != nil`; standard TLS termination normally forwards HTTP with `r.TLS == nil`. No trusted-forwarded-proto handling exists. Thus cookies can lack the Secure flag behind a conventional proxy. This is a direct code fact; severity depends on proxy/network configuration.
6. **Medium: installer command generation trusts request `Host` without host validation.** `handleCreateEnrollToken` and `handleInstallScript` interpolate `r.Host` into shell/PowerShell script content. Tokens are sanitized, but host is not. This permits reflected malicious installer content to callers who can control Host; impact depends on whether an attacker can induce a target to fetch/use that response. Use an explicit configured public base URL or strict allowlist.
7. **Medium: update integrity stops at a mutable GitHub checksum manifest.** `internal/updater/updater.go` validates checksum supplied by `checksums.txt` and binary version, but does not validate release provenance/signature/attestation despite release workflow generating an attestation. A GitHub repository/release-distribution compromise can publish a matching altered binary and manifest.
8. **Medium: API/MCP file tools claim broad cross-platform operation but depend on Unix commands.** MCP `ls`, `cat`, and `base64 -d` will typically fail on native Windows. This is a capability/reliability issue which can yield unsafe model workarounds (`run_command`) rather than a direct injection; shell quoting of path/content is otherwise intentionally careful.
9. **Medium: no separate CSRF protection on state-changing cookie-authenticated API routes.** SameSite=Lax materially helps ordinary cross-site form/subresource requests, JSON content type prevents simple-form JSON calls, and WebSocket origin checks help WS routes. However no CSRF token/origin middleware exists for mutations; deployment behind unusual same-site/subdomain setups should not assume it is a complete CSRF model.
10. **Low/medium availability: upload intake is not protected by an explicit total `MaxBytesReader` limit.** `handleFsUpload` uses `ParseMultipartForm(32 << 20)`, which limits in-memory form handling but does not clearly impose the same hard total request cap used elsewhere; an authenticated client may cause temporary disk/memory pressure. Enforce a total body limit and stream accounting.
11. **Low: browser sessions are in-memory only.** A hub restart invalidates them despite a 30-day cookie `MaxAge`. This is not a confidentiality flaw, but can cause unexpected lockout/operational disruption and explains why cookie persistence is only apparent.

### Recommendations (not claims of existing vulnerabilities)

Prioritize mandatory initial local/loopback bootstrap or a startup one-time secret; TLS-by-default or refusal to bind publicly without explicit insecure acknowledgement; scoped/expiring/revocable API tokens (per device + read/terminal/file/exec roles); operator-visible command/audit logs; explicit proxy trust/public URL configuration; file-size and transfer quotas; native file MCP APIs rather than shell utilities; and signed/provenance-verified release artifacts. Add per-device agent identity rotation and an allowlist/project-sandbox option for less trusted agents.

## Architecture maps

### Human control path

```text
User browser
  -> React UI (`ui/`, cookie-authenticated same-origin REST/WebSocket)
  -> Hub (`cmd/overseer`, `internal/hub`)
      -> SQLite for metadata/tokens/projects
      -> live `agentConn` registry
  -> one authenticated outbound WebSocket (`internal/protocol`)
  -> Agent (`internal/agent`) running as its service user
  -> tmux/PTy, shell/PowerShell, filesystem, and device OS
```

Browser terminal bytes take a second multiplexed branch: xterm WebSocket `/api/ws/term` → hub-generated channel → agent `term.open`/binary PTY stream → tmux/ConPTY. Device stats travel agent → hub event bus → `/api/ws/events` → UI.

### AI control path

```text
AI agent / MCP client
  -> stdio `overseer mcp` + saved fleet token
     OR HTTPS POST `/mcp` + Bearer API token
  -> `internal/mcp` tool dispatcher
  -> `internal/fleet.Client` REST calls to hub
  -> normal authenticated hub handlers
  -> device agent outbound WebSocket
  -> command/session/file operation on device
```

MCP has no extra authorization layer beyond the same API token. The remote HTTP endpoint internally re-enters the hub through a loopback listener to reuse the normal handler path.

## Safest Salmon integration points, based only on existing code

Current Salmon should **not** be wired directly to agent WebSockets or granted special agent credentials. The safest existing integration boundary is the already-authenticated fleet/MCP abstraction:

1. Add a deliberate Salmon command/service adapter at `cmd/overseer/main.go` only after configuration/auth design is chosen. Construct Salmon with dependency injection rather than hard-coded environment/default hub values.
2. Give Salmon fleet action capability through `internal/fleet.Client` (typed API functions) or the `internal/mcp` tool dispatch surface—not `internal/hub.registry`, `agentConn`, or raw protocol. These boundaries already use API-token authentication and central hub validations.
3. If a browser Salmon interface is added, add a new authenticated route/handler in `internal/hub/` protected by `requireAuth`, and a page/component under `ui/src/pages/` / `ui/src/components/`. The handler should call a narrowly defined Salmon service and pass a scoped token/client; do not expose Gemini keys or raw fleet credentials to the UI.
4. Store only non-secret Salmon settings/project metadata in `Store` after an explicit schema decision. Keep provider keys in environment/OS secret storage; do not add them to SQLite settings as plaintext. Token scope, device allowlisting, command confirmation, activity/audit trail, cancellation, and streaming behavior need design before a command-capable Salmon agent is enabled.
5. A remote Salmon endpoint, if ever needed, should be a separate explicitly permissioned MCP/API tool rather than silently adding behavior to `/mcp`, because `/mcp` already represents unrestricted remote machine control.

## Missing, incomplete, stale, or experimental pieces

- `internal/salmon/salmon.go` is untracked, not imported, has no UI/route/CLI, never uses its Overseer fields, and explicitly says execution is future work.
- The approved design specification is stale in several implementation details, including Windows and older endpoint/tool descriptions.
- Native Windows support is substantially implemented for agent enrollment/service/ConPTY, but tmux-backed persistence remains unavailable and MCP file helpers are Unix-specific. REST `send_input`/`read_output` themselves hard-code tmux, so they do not support ephemeral Windows/non-tmux sessions despite those sessions being representable in listing/terminal paths.
- UI and marketing site have no automated test suites; site is not built/linted by root CI.
- No multi-user/team model, scopes/roles, audit trail, token expiration, device approval/rotation, native mobile app, or built-in tunnel exists.
- No full MCP protocol/session/SSE implementation; only a tools-only POST JSON-RPC subset is implemented.
- Session status is a 10-second tmux activity heuristic, not semantic coding-agent state; sessions are not durable on native Windows or when tmux is unavailable.
- Setup authentication status is only provider CLI probing, not a robust credential verification system. Setup commands necessarily execute third-party installers as the device service user.
- Update UI status may show release information but full replacement only works in a managed-service installation; release provenance exists in CI but is not verified at runtime.
- Public static installer copies in `site/public/` duplicate root scripts exactly now, creating a maintenance drift surface.

## Final summary and recommended next steps

Critical code starts at `cmd/overseer/main.go`, `internal/hub/server.go`, `internal/hub/api.go`, `internal/hub/auth.go`, `internal/hub/ws.go`, `internal/agent/agent.go`, `internal/agent/ops.go`, `internal/protocol/protocol.go`, and `internal/mcp/mcp.go`. Critical dependencies are Gorilla WebSocket, gopsutil, modernc SQLite, Argon2/autocert from `x/crypto`, PTY/ConPTY, React/xterm/libfx, Vite, and Tailwind.

Current capabilities are real fleet enrollment, reverse/outbound device connectivity, live terminal streaming, tmux sessions, commands, file transfer/browser, device telemetry, fixed tool setup, projects/fx command routing, fleet CLI, local/remote MCP, update/rollback, and multi-platform installers. The architecture is compact and coherent: hub-and-spoke WebSocket multiplexing keeps remote devices NAT-friendly, and protocol/agent/hub responsibilities are clearly separated.

Current limitations are chiefly security granularity and maturity: no sandboxing/scopes/multi-user model, HTTP default, setup-bootstrap exposure, lack of complete Windows parity, isolated Salmon, limited MCP transport/tool portability, and absent frontend/site/security testing.

Recommended implementation order:

1. Fix bootstrap/public-TLS/token-scope boundaries before adding AI automation: require safe first-run setup, strongly discourage/guard public HTTP, correct proxy cookie handling, and add expiring scoped per-device tokens.
2. Add audit logs, command confirmations/policy hooks, device token rotation, transfer quotas, and test security/stream error paths.
3. Make file/session control platform-native (especially Windows) and add full UI/E2E and installer/update coverage.
4. Design Salmon as a separately permissioned service over `fleet.Client`/MCP-level operations, then add its authenticated UI/API and provider-key management—never raw protocol access or a browser-visible secret.
