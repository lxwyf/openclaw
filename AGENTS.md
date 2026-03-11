# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Quick Reference

- **Repo**: https://github.com/openclaw/openclaw
- **Version**: 2026.2.4 (dev) / see `package.json` and `CHANGELOG.md`
- **Node baseline**: ≥22.12.0
- **Package manager**: pnpm (also supports bun for scripts/dev; Node for production)
- **Language**: TypeScript ESM, strict typing
- **GitHub**: Use literal multiline strings or `-F - <<'EOF'` (or `$'...'`) for real newlines; never embed `\\n`

## High-Level Architecture

OpenClaw is a **personal AI assistant platform** with a WebSocket-based control plane (Gateway) and multiple client surfaces:

```
┌─────────────────────────────────────────────────────────────────┐
│ Gateway (WebSocket @ 127.0.0.1:18789)                           │
│ ├─ Message routing (channels → agents)                           │
│ ├─ Session management (per-user/group/workspace isolation)       │
│ ├─ Device pairing (iOS/Android/macOS nodes)                      │
│ ├─ Event broadcasting (tool calls, thinking, streams)            │
│ └─ HTTP web surfaces (Dashboard, WebChat, Canvas)                │
└─────────────────────────────────────────────────────────────────┘
     ↑                                ↑                        ↑
     │                                │                        │
  CLI/Dev    Mobile/macOS         External           (Extensions)
  Commands   Nodes (Canvas,        Messaging         Channel
             Voice Wake)           Channels          Plugins
```

### Core Abstractions

1. **Agents** (`src/agents/`) - 297 files, AI execution engine
   - Lifecycle: configuration → workspace → model selection → tool execution → response handling
   - Pi-mono integration (0.52.5): RPC-mode AI agent with streaming blocks
   - Multi-agent support: main agent + configurable subagents (e.g., research, coding)
   - Tool system: unified API for bash/files/messaging/channel-specific operations
   - Context management: automatic compaction, token budget enforcement, lane-based concurrency

2. **Channels** (`src/{telegram,discord,slack,whatsapp,signal,imessage,web}/`) - 12+ platforms
   - Adapter pattern: each channel implements common interface (send/receive/react/etc.)
   - Extension channels in `extensions/*` (BlueBubbles, Teams, Matrix, Zalo, etc.)
   - Unified routing: messages flow through `src/routing/` to agents regardless of source
   - Message metadata preservation (sender, group, thread, media links)

3. **Gateway** (`src/gateway/`) - WebSocket control plane
   - Client connect/auth → device pairing → message dispatch → event broadcast
   - Stateless transport; device tokens stored in `~/.openclaw/nodes/paired.json`
   - Supports local (loopback), LAN (Bonjour), and remote (Tailscale) access
   - Event system: `device.pair.requested`, `message.incoming`, `tool.call`, etc.

4. **Configuration** (`src/config/`) - JSON5-based, environment-aware
   - Location: `~/.openclaw/openclaw.json`
   - Supports multi-agent, per-channel overrides, environment variable injection
   - Validation via Zod schemas
   - Hot-reload: config changes apply without gateway restart (mostly)

5. **Workspace** (`src/agents/`) - persistent agent state
   - Location: `~/.openclaw/workspace` or `~/.openclaw/workspace-{agentId}`
   - Files: `AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `MEMORY.md`, `BOOTSTRAP.md`
   - Acts as agent's "filesystem brain" – referenced in system prompts
   - Supports git integration for version control

### Request Flow Example

```
User sends WhatsApp message
  ↓
Baileys (WhatsApp adapter) parses it
  ↓
src/routing/router.ts routes by channel/group/user
  ↓
Resolves agent (config.agents) + session scope
  ↓
src/agents/pi-embedded-runner/run.ts executes:
  1. Load workspace files → system prompt
  2. Resolve model + auth (with failover)
  3. Fetch session history (optional compaction)
  4. Stream through Pi-mono runtime
  5. Execute tool calls (bash/files/message/etc.)
  ↓
Response blocks collected, deduplicated, chunked
  ↓
Message tool or direct reply back to WhatsApp
```

### Key Design Patterns

- **Policy pattern**: Tool access, DM scope, group policies all configurable
- **Lane-based queueing**: `src/agents/lanes.ts` – per-session lanes + global lane prevent race conditions
- **Streaming architecture**: Message blocks streamed incrementally; clients subscribe for real-time updates
- **Failover cascading**: Auth failure → try next API key/provider; context overflow → compress history
- **Workspace isolation**: Each agent gets its own directory; multi-workspace supports >1 bot per machine

---

## Build, Test, and Development

### Common Commands

```bash
# Install & setup
pnpm install
pnpm ui:build                           # Build Web UI assets
pnpm build                              # Full build: TS compile + UI copy + metadata

# Development
pnpm dev                                # Run CLI in dev mode (pnpm openclaw ...)
pnpm gateway:watch                      # Watch gateway, auto-restart on changes
pnpm openclaw <cmd>                     # Run CLI commands directly

# Type-check & lint
pnpm tsgo                               # Quick type check (no emit)
pnpm check                              # Full gate: tsgo + lint + format
pnpm lint                               # Oxlint only
pnpm format                             # Oxfmt only

# Testing
pnpm test                               # Run all tests (Vitest, parallel)
pnpm test:coverage                      # Coverage report (70% threshold: lines/branches/functions/statements)
pnpm test -- src/agents/model-auth.test.ts  # Run single test file
pnpm test -- --run src/agents/model-auth.test.ts  # Single test, no watch
pnpm test:live                          # Live tests (needs LIVE=1 or real API keys)
pnpm test:docker:*                      # E2E Docker tests (gateway, onboarding, models)

# Platform-specific builds
pnpm mac:package                        # Build macOS app
pnpm ios:run                            # Build & run iOS simulator
pnpm android:run                        # Build & run Android emulator

# UI & docs
pnpm ui:dev                             # Dev server for Web UI (hot reload)
pnpm docs:bin                           # Generate CLI command reference
pnpm canvas:a2ui:bundle                 # Rebuild A2UI Canvas bundles
```

### Full Pre-Commit Gate

Before pushing or committing:

```bash
pnpm build && pnpm check && pnpm test
```

If formatting-only diffs appear, auto-resolve (staged + unstaged):

```bash
pnpm format
git add <affected-files>
```

### One-Off Development Patterns

**Run a single Agent test file**:
```bash
pnpm test -- src/agents/model-selection.test.ts --run
```

**Debug Pi agent execution locally**:
```bash
# 1. Add logging in src/agents/pi-embedded-runner/run.ts
// logDebug({ runAttempt, sessionMessages, toolResults });

# 2. Run test with stdio
pnpm test -- src/agents/pi-embedded-runner.test.ts --reporter=verbose --reporter=default

# 3. Or run CLI directly
pnpm openclaw agent --message "hello"
```

**Rebuild and test a specific channel**:
```bash
pnpm build
pnpm openclaw channels status --probe  # Verify connectivity
pnpm test -- src/telegram/  # Run Telegram tests
```

**Interactive config editing**:
```bash
openclaw config set channels.telegram.dm.policy open
openclaw config get channels.telegram.dm
openclaw config set gateway.bind 0.0.0.0  # Remote access
```

---

## Repository Guidelines

## Project Structure & Module Organization

### Directoy Layout

- `src/` - Main source code
  - `agents/` (297 files, 46K+ LOC) – AI execution engine: Pi-mono integration, tool system, model auth/failover, session management, workspace lifecycle
  - `cli/` – Command parsing, help, progress spinners
  - `commands/` – Individual commands (agent, message, channels, config, etc.)
  - `gateway/` – WebSocket server, client protocol, message dispatch, device pairing
  - `{telegram,discord,slack,whatsapp,signal,imessage,web}/` – Channel adapters (unified interface)
  - `channels/` – Shared routing, message types, media handling
  - `routing/` – Message routing logic (which agent, which session scope)
  - `config/` – Configuration loading, validation (Zod), environment substitution
  - `sessions/` – Session state management and persistence
  - `infra/` – OS integration (binaries, ports, environment)
  - `media/` – Media processing (images, audio, video, transcription hooks)
  - `browser/` – Playwright-based browser control for tools
  - `cron/` – Scheduled tasks and automation
  - `terminal/` – CLI UI (tables, spinners, palette)
  - `web/` – HTTP server for Dashboard/WebChat/Canvas (NextJS-based)
  - `plugin-sdk/` – Plugin system SDK
  - `security/`, `pairing/`, `memory/` – Domain-specific modules

- `extensions/` – Official channel plugins (30+): BlueBubbles, Teams, Matrix, Zalo, Voice Call, etc.
  - Each has own `package.json` (no `workspace:*` in deps); uses jiti alias to import `openclaw/plugin-sdk`
  - Install runs `npm install --omit=dev` in plugin dir

- `apps/` – Platform applications
  - `macos/` – SwiftUI macOS app (voice wake, menubar, control)
  - `ios/` – Swift iOS app (nodes, canvas, voice wake)
  - `android/` – Kotlin Android app

- `docs/` – Mintlify documentation (root-relative links, no `.md` extension)
  - `zh-CN/` – Auto-generated Chinese translations; do not edit unless explicitly asked
  - `.i18n/` – Translation glossary, memory, pipeline config

- `skills/` – Built-in skill library (bundled with release)

- `scripts/` – Build, utility, and release scripts (see `scripts/committer` for commits)

- Tests: colocated as `*.test.ts` and `*.e2e.test.ts`
- Built output: `dist/`

### When Refactoring Shared Logic

**Messaging channels**: always consider **all** built-in + extension channels. Review code in:
- Core channels: `src/{telegram,discord,slack,signal,imessage,web}` + `src/routing`
- Extensions: `extensions/*` (may implement same interface differently)
- Docs: `docs/channels/` + provider-specific routing/pairing/command docs
- Label coverage: check `.github/labeler.yml` when adding new channels/extensions

Example refactor scope: DM pairing policy touches routing, all channel adapters, config validation, CLI, docs, and extension channels.

### External Dependencies

- **Installers**: served from `https://openclaw.ai/*`, live in sibling repo `../openclaw.ai` (`public/install.sh`, `public/install-cli.sh`, `public/install.ps1`)
- **Extensions**: keep plugin-only deps in extension `package.json`; do not add to root unless core uses them
  - Avoid `workspace:*` in extension `dependencies` (breaks npm install); put `openclaw` in `devDependencies` or `peerDependencies` instead
  - Runtime resolves `openclaw/plugin-sdk` via jiti alias

## Agent Module Deep Dive

The Agent module (`src/agents/`) is the execution engine. Understanding its flow is critical for most changes:

### Configuration → Execution Flow

```
1. AgentScope Resolution (agent-scope.ts)
   ├─ Load openclaw.json
   ├─ Resolve agent ID (from config or CLI)
   └─ Extract agent-specific config (model, tools, workspace, identity)

2. Workspace Initialization (workspace.ts)
   ├─ Location: ~/.openclaw/workspace (or ~/.openclaw/workspace-{agentId})
   ├─ Load AGENTS.md, SOUL.md, TOOLS.md, IDENTITY.md, MEMORY.md
   └─ Files become system prompt context

3. Model Resolution (model-selection.ts + model-auth.ts)
   ├─ Parse "anthropic/claude-opus-4-6" format
   ├─ Resolve auth (API key from env, config, or auth profile store)
   ├─ Support auth profile rotation (OAuth, API keys, AWS SDK)
   └─ Auth failures trigger model failover (cascade to next provider/key)

4. Pi-Embedded Runner (pi-embedded-runner/run.ts)
   ├─ Create session lane (per-sessionKey concurrency control)
   ├─ Build system prompt (system-prompt.ts) with workspace context
   ├─ Resolve tools (pi-tools.ts: read/write/exec/message/etc., filtered by policy)
   ├─ Fetch session history (with optional auto-compaction)
   ├─ Call Pi-mono runtime in RPC mode
   └─ Stream response blocks with tool call handling

5. Tool Execution (bash-tools.exec.ts, pi-tools.ts)
   ├─ Tool policies enforce allowlist/denylists
   ├─ Bash commands: security check → approve flow → PTY/process exec
   ├─ File operations: read/write/apply_patch with sandbox checks
   ├─ Message tool: route across channels (Telegram, Discord, etc.)
   └─ Deduplicate messaging to prevent duplicate sends

6. Response Handling (pi-embedded-subscribe.ts)
   ├─ Collect text blocks (streaming or final)
   ├─ Deduplicate against recent messages
   ├─ Split into chunks (text_end, tool_call breakpoints)
   ├─ Broadcast events to web UI / nodes
   └─ Deliver final reply to source channel
```

### Key Patterns

**Lane-based Concurrency**:
- `src/agents/lanes.ts`: per-session lanes + global lane queue
- Prevents race conditions in tool execution and session updates
- Example: two messages to same DM scope won't execute simultaneously

**Auth Profile Rotation**:
- `src/agents/auth-profiles/` + `auth-profiles.ts`
- Multiple API keys per provider; automatic rotation on failure
- Cooldown periods to avoid hammering failed keys
- Last-good profile prioritized

**Context Window Management**:
- `context-window-guard.ts`: enforce token budget
- `compaction.ts`: auto-prune history when approaching limit
- Bootstrap files injected first to ensure workspace context

**Tool Filtering by Policy**:
- `pi-tools.ts`: each tool has name (e.g., "exec", "read", "message")
- `tool-policy.ts`: resolve allowlist (supports wildcards "exec*")
- Subagents get stricter policies (read-only tools by default)

### Testing Agent Changes

```bash
# Test model selection & auth
pnpm test -- src/agents/model-selection.test.ts

# Test auth profile rotation
pnpm test -- src/agents/auth-profiles.resolve-auth-profile-order.uses-stored-profiles-no-config-exists.test.ts

# Test Pi execution (mocked)
pnpm test -- src/agents/pi-embedded-runner.test.ts

# Test tool policy filtering
pnpm test -- src/agents/pi-tools.policy.ts

# Test bash tool execution
pnpm test -- src/agents/bash-tools.exec.ts

# Live test (needs Anthropic API key)
CLAWDBOT_LIVE_TEST=1 pnpm test:live -- src/agents/
```

### Common Agent-Related Tasks

**Add a new tool**:
1. Define in `pi-tools.ts` (schema + handler)
2. Add to tool policy docs
3. Test with `pnpm test -- src/agents/pi-tools.schema.ts`
4. Update system prompt hints in `system-prompt.ts`

**Add workspace file support**:
1. Define path in `workspace.ts`
2. Inject into system prompt in `system-prompt.ts`
3. Document in `docs/concepts/workspace.md`

**Modify agent execution flow**:
1. Start in `pi-embedded-runner/run.ts` (entry point)
2. Check lane queueing in `lanes.ts`
3. System prompt generation: `system-prompt.ts`
4. Tool execution loop: `pi-embedded-subscribe.ts`

---

## Messaging Channel Development

When adding or modifying a messaging channel:

### Channel Architecture

Each channel implements a common interface:
```typescript
// Location varies: src/telegram/*, src/discord/*, extensions/*/
export interface ChannelAdapter {
  send(message, options?) => Promise<SendResult>;
  receive(update) => Promise<ReceivedMessage>;
  react(messageId, reaction) => Promise<void>;
  // + platform-specific methods
}
```

### Integration Points (Checklist)

- [ ] **Config schema** (`src/config/types.{channel}.ts`): Define config structure
- [ ] **Routing** (`src/routing/router.ts`): Map messages to agents/sessions
- [ ] **Channel adapter** (`src/{channel}/` or `extensions/{channel}/`): Implement send/receive/etc.
- [ ] **Message types** (`src/channels/message-types.ts`): Define metadata (sender, group, media)
- [ ] **Media handling** (`src/media/`): Handle downloads/uploads
- [ ] **DM policy** (`src/config/types.base.ts`): Support DmPolicy ("open", "pairing", "allowlist", "disabled")
- [ ] **Group policy** (`src/config/types.base.ts`): Support GroupPolicy ("open", "allowlist", "disabled")
- [ ] **Command gating** (`src/routing/`): Filter commands by channel
- [ ] **Onboarding** (`src/wizard/`): Add channel setup flow
- [ ] **CLI commands** (`src/commands/{channel}.ts`): Add manage/login/etc.
- [ ] **Docs** (`docs/channels/{channel}.md`): Document setup, features, limitations
- [ ] **Labels** (`.github/labeler.yml`): Add label for auto-categorization
- [ ] **Extension**: If not in `src/`, ensure `extensions/{channel}/` follows plugin patterns

### Quick Start: Add a Telegram Command

```bash
# 1. Implement handler in src/telegram/bot-handlers.ts
export async function handleCommand(msg, context) {
  if (msg.text?.startsWith("/mycommand")) {
    // your logic
  }
}

# 2. Add to command registry
// In bot.ts or bot-handlers.ts
bot.command("mycommand", handleCommand);

# 3. Test
pnpm test -- src/telegram/

# 4. Update docs
# docs/channels/telegram.md → add command reference
```

---
- Internal doc links in `docs/**/*.md`: root-relative, no `.md`/`.mdx` (example: `[Config](/configuration)`).
- Section cross-references: use anchors on root-relative paths (example: `[Hooks](/configuration#hooks)`).
- Doc headings and anchors: avoid em dashes and apostrophes in headings because they break Mintlify anchor links.
- When Peter asks for links, reply with full `https://docs.openclaw.ai/...` URLs (not root-relative).
- When you touch docs, end the reply with the `https://docs.openclaw.ai/...` URLs you referenced.
- README (GitHub): keep absolute docs URLs (`https://docs.openclaw.ai/...`) so links work on GitHub.
- Docs content must be generic: no personal device names/hostnames/paths; use placeholders like `user@gateway-host` and “gateway host”.

## Docs i18n (zh-CN)

- `docs/zh-CN/**` is generated; do not edit unless the user explicitly asks.
- Pipeline: update English docs → adjust glossary (`docs/.i18n/glossary.zh-CN.json`) → run `scripts/docs-i18n` → apply targeted fixes only if instructed.
- Translation memory: `docs/.i18n/zh-CN.tm.jsonl` (generated).
- See `docs/.i18n/README.md`.
- The pipeline can be slow/inefficient; if it’s dragging, ping @jospalmbier on Discord instead of hacking around it.

## exe.dev VM ops (general)

- Access: stable path is `ssh exe.dev` then `ssh vm-name` (assume SSH key already set).
- SSH flaky: use exe.dev web terminal or Shelley (web agent); keep a tmux session for long ops.
- Update: `sudo npm i -g openclaw@latest` (global install needs root on `/usr/lib/node_modules`).
- Config: use `openclaw config set ...`; ensure `gateway.mode=local` is set.
- Discord: store raw token only (no `DISCORD_BOT_TOKEN=` prefix).
- Restart: stop old gateway and run:
  `pkill -9 -f openclaw-gateway || true; nohup openclaw gateway run --bind loopback --port 18789 --force > /tmp/openclaw-gateway.log 2>&1 &`
- Verify: `openclaw channels status --probe`, `ss -ltnp | rg 18789`, `tail -n 120 /tmp/openclaw-gateway.log`.

## Build, Test, and Development Commands

- Runtime baseline: Node **22+** (keep Node + Bun paths working).
- Install deps: `pnpm install`
- Pre-commit hooks: `prek install` (runs same checks as CI)
- Also supported: `bun install` (keep `pnpm-lock.yaml` + Bun patching in sync when touching deps/patches).
- Prefer Bun for TypeScript execution (scripts, dev, tests): `bun <file.ts>` / `bunx <tool>`.
- Run CLI in dev: `pnpm openclaw ...` (bun) or `pnpm dev`.
- Node remains supported for running built output (`dist/*`) and production installs.
- Mac packaging (dev): `scripts/package-mac-app.sh` defaults to current arch. Release checklist: `docs/platforms/mac/release.md`.
- Type-check/build: `pnpm build`
- TypeScript checks: `pnpm tsgo`
- Lint/format: `pnpm check`
- Tests: `pnpm test` (vitest); coverage: `pnpm test:coverage`

## Coding Style & Naming Conventions

- Language: TypeScript (ESM). Prefer strict typing; avoid `any`.
- Formatting/linting via Oxlint and Oxfmt; run `pnpm check` before commits.
- Add brief code comments for tricky or non-obvious logic.
- Keep files concise; extract helpers instead of “V2” copies. Use existing patterns for CLI options and dependency injection via `createDefaultDeps`.
- Aim to keep files under ~700 LOC; guideline only (not a hard guardrail). Split/refactor when it improves clarity or testability.
- Naming: use **OpenClaw** for product/app/docs headings; use `openclaw` for CLI command, package/binary, paths, and config keys.

## Release Channels (Naming)

- stable: tagged releases only (e.g. `vYYYY.M.D`), npm dist-tag `latest`.
- beta: prerelease tags `vYYYY.M.D-beta.N`, npm dist-tag `beta` (may ship without macOS app).
- dev: moving head on `main` (no tag; git checkout main).

## Testing Guidelines

- Framework: Vitest with V8 coverage thresholds (70% lines/branches/functions/statements).
- Naming: match source names with `*.test.ts`; e2e in `*.e2e.test.ts`.
- Run `pnpm test` (or `pnpm test:coverage`) before pushing when you touch logic.
- Do not set test workers above 16; tried already.
- Live tests (real keys): `CLAWDBOT_LIVE_TEST=1 pnpm test:live` (OpenClaw-only) or `LIVE=1 pnpm test:live` (includes provider live tests). Docker: `pnpm test:docker:live-models`, `pnpm test:docker:live-gateway`. Onboarding Docker E2E: `pnpm test:docker:onboard`.
- Full kit + what’s covered: `docs/testing.md`.
- Pure test additions/fixes generally do **not** need a changelog entry unless they alter user-facing behavior or the user asks for one.
- Mobile: before using a simulator, check for connected real devices (iOS + Android) and prefer them when available.

## Commit & Pull Request Guidelines

- Create commits with `scripts/committer "<msg>" <file...>`; avoid manual `git add`/`git commit` so staging stays scoped.
- Follow concise, action-oriented commit messages (e.g., `CLI: add verbose flag to send`).
- Group related changes; avoid bundling unrelated refactors.
- Changelog workflow: keep latest released version at top (no `Unreleased`); after publishing, bump version and start a new top section.
- PRs should summarize scope, note testing performed, and mention any user-facing changes or new flags.
- Read this when submitting a PR: `docs/help/submitting-a-pr.md` ([Submitting a PR](https://docs.openclaw.ai/help/submitting-a-pr))
- Read this when submitting an issue: `docs/help/submitting-an-issue.md` ([Submitting an Issue](https://docs.openclaw.ai/help/submitting-an-issue))
- PR review flow: when given a PR link, review via `gh pr view`/`gh pr diff` and do **not** change branches.
- PR review calls: prefer a single `gh pr view --json ...` to batch metadata/comments; run `gh pr diff` only when needed.
- Before starting a review when a GH Issue/PR is pasted: run `git pull`; if there are local changes or unpushed commits, stop and alert the user before reviewing.
- Goal: merge PRs. Prefer **rebase** when commits are clean; **squash** when history is messy.
- PR merge flow: create a temp branch from `main`, merge the PR branch into it (prefer squash unless commit history is important; use rebase/merge when it is). Always try to merge the PR unless it’s truly difficult, then use another approach. If we squash, add the PR author as a co-contributor. Apply fixes, add changelog entry (include PR # + thanks), run full gate before the final commit, commit, merge back to `main`, delete the temp branch, and end on `main`.
- If you review a PR and later do work on it, land via merge/squash (no direct-main commits) and always add the PR author as a co-contributor.
- When working on a PR: add a changelog entry with the PR number and thank the contributor.
- When working on an issue: reference the issue in the changelog entry.
- When merging a PR: leave a PR comment that explains exactly what we did and include the SHA hashes.
- When merging a PR from a new contributor: add their avatar to the README “Thanks to all clawtributors” thumbnail list.
- After merging a PR: run `bun scripts/update-clawtributors.ts` if the contributor is missing, then commit the regenerated README.

## Shorthand Commands

- `sync`: if working tree is dirty, commit all changes (pick a sensible Conventional Commit message), then `git pull --rebase`; if rebase conflicts and cannot resolve, stop; otherwise `git push`.

### PR Workflow (Review vs Land)

- **Review mode (PR link only):** read `gh pr view/diff`; **do not** switch branches; **do not** change code.
- **Landing mode:** create an integration branch from `main`, bring in PR commits (**prefer rebase** for linear history; **merge allowed** when complexity/conflicts make it safer), apply fixes, add changelog (+ thanks + PR #), run full gate **locally before committing** (`pnpm build && pnpm check && pnpm test`), commit, merge back to `main`, then `git switch main` (never stay on a topic branch after landing). Important: contributor needs to be in git graph after this!

## Security & Configuration Tips

- Web provider stores creds at `~/.openclaw/credentials/`; rerun `openclaw login` if logged out.
- Pi sessions live under `~/.openclaw/sessions/` by default; the base directory is not configurable.
- Environment variables: see `~/.profile`.
- Never commit or publish real phone numbers, videos, or live configuration values. Use obviously fake placeholders in docs, tests, and examples.
- Release flow: always read `docs/reference/RELEASING.md` and `docs/platforms/mac/release.md` before any release work; do not ask routine questions once those docs answer them.

## Troubleshooting

- Rebrand/migration issues or legacy config/service warnings: run `openclaw doctor` (see `docs/gateway/doctor.md`).

## Agent-Specific Notes

- Vocabulary: "makeup" = "mac app".
- Never edit `node_modules` (global/Homebrew/npm/git installs too). Updates overwrite. Skill notes go in `tools.md` or `AGENTS.md`.
- Signal: "update fly" => `fly ssh console -a flawd-bot -C "bash -lc 'cd /data/clawd/openclaw && git pull --rebase origin main'"` then `fly machines restart e825232f34d058 -a flawd-bot`.
- When working on a GitHub Issue or PR, print the full URL at the end of the task.
- When answering questions, respond with high-confidence answers only: verify in code; do not guess.
- Never update the Carbon dependency.
- Any dependency with `pnpm.patchedDependencies` must use an exact version (no `^`/`~`).
- Patching dependencies (pnpm patches, overrides, or vendored changes) requires explicit approval; do not do this by default.
- CLI progress: use `src/cli/progress.ts` (`osc-progress` + `@clack/prompts` spinner); don’t hand-roll spinners/bars.
- Status output: keep tables + ANSI-safe wrapping (`src/terminal/table.ts`); `status --all` = read-only/pasteable, `status --deep` = probes.
- Gateway currently runs only as the menubar app; there is no separate LaunchAgent/helper label installed. Restart via the OpenClaw Mac app or `scripts/restart-mac.sh`; to verify/kill use `launchctl print gui/$UID | grep openclaw` rather than assuming a fixed label. **When debugging on macOS, start/stop the gateway via the app, not ad-hoc tmux sessions; kill any temporary tunnels before handoff.**
- macOS logs: use `./scripts/clawlog.sh` to query unified logs for the OpenClaw subsystem; it supports follow/tail/category filters and expects passwordless sudo for `/usr/bin/log`.
- If shared guardrails are available locally, review them; otherwise follow this repo's guidance.
- SwiftUI state management (iOS/macOS): prefer the `Observation` framework (`@Observable`, `@Bindable`) over `ObservableObject`/`@StateObject`; don’t introduce new `ObservableObject` unless required for compatibility, and migrate existing usages when touching related code.
- Connection providers: when adding a new connection, update every UI surface and docs (macOS app, web UI, mobile if applicable, onboarding/overview docs) and add matching status + configuration forms so provider lists and settings stay in sync.
- Version locations: `package.json` (CLI), `apps/android/app/build.gradle.kts` (versionName/versionCode), `apps/ios/Sources/Info.plist` + `apps/ios/Tests/Info.plist` (CFBundleShortVersionString/CFBundleVersion), `apps/macos/Sources/OpenClaw/Resources/Info.plist` (CFBundleShortVersionString/CFBundleVersion), `docs/install/updating.md` (pinned npm version), `docs/platforms/mac/release.md` (APP_VERSION/APP_BUILD examples), Peekaboo Xcode projects/Info.plists (MARKETING_VERSION/CURRENT_PROJECT_VERSION).
- **Restart apps:** “restart iOS/Android apps” means rebuild (recompile/install) and relaunch, not just kill/launch.
- **Device checks:** before testing, verify connected real devices (iOS/Android) before reaching for simulators/emulators.
- iOS Team ID lookup: `security find-identity -p codesigning -v` → use Apple Development (…) TEAMID. Fallback: `defaults read com.apple.dt.Xcode IDEProvisioningTeamIdentifiers`.
- A2UI bundle hash: `src/canvas-host/a2ui/.bundle.hash` is auto-generated; ignore unexpected changes, and only regenerate via `pnpm canvas:a2ui:bundle` (or `scripts/bundle-a2ui.sh`) when needed. Commit the hash as a separate commit.
- Release signing/notary keys are managed outside the repo; follow internal release docs.
- Notary auth env vars (`APP_STORE_CONNECT_ISSUER_ID`, `APP_STORE_CONNECT_KEY_ID`, `APP_STORE_CONNECT_API_KEY_P8`) are expected in your environment (per internal release docs).
- **Multi-agent safety:** do **not** create/apply/drop `git stash` entries unless explicitly requested (this includes `git pull --rebase --autostash`). Assume other agents may be working; keep unrelated WIP untouched and avoid cross-cutting state changes.
- **Multi-agent safety:** when the user says "push", you may `git pull --rebase` to integrate latest changes (never discard other agents' work). When the user says "commit", scope to your changes only. When the user says "commit all", commit everything in grouped chunks.
- **Multi-agent safety:** do **not** create/remove/modify `git worktree` checkouts (or edit `.worktrees/*`) unless explicitly requested.
- **Multi-agent safety:** do **not** switch branches / check out a different branch unless explicitly requested.
- **Multi-agent safety:** running multiple agents is OK as long as each agent has its own session.
- **Multi-agent safety:** when you see unrecognized files, keep going; focus on your changes and commit only those.
- Lint/format churn:
  - If staged+unstaged diffs are formatting-only, auto-resolve without asking.
  - If commit/push already requested, auto-stage and include formatting-only follow-ups in the same commit (or a tiny follow-up commit if needed), no extra confirmation.
  - Only ask when changes are semantic (logic/data/behavior).
- Lobster seam: use the shared CLI palette in `src/terminal/palette.ts` (no hardcoded colors); apply palette to onboarding/config prompts and other TTY UI output as needed.
- **Multi-agent safety:** focus reports on your edits; avoid guard-rail disclaimers unless truly blocked; when multiple agents touch the same file, continue if safe; end with a brief “other files present” note only if relevant.
- Bug investigations: read source code of relevant npm dependencies and all related local code before concluding; aim for high-confidence root cause.
- Code style: add brief comments for tricky logic; keep files under ~500 LOC when feasible (split/refactor as needed).
- Tool schema guardrails (google-antigravity): avoid `Type.Union` in tool input schemas; no `anyOf`/`oneOf`/`allOf`. Use `stringEnum`/`optionalStringEnum` (Type.Unsafe enum) for string lists, and `Type.Optional(...)` instead of `... | null`. Keep top-level tool schema as `type: "object"` with `properties`.
- Tool schema guardrails: avoid raw `format` property names in tool schemas; some validators treat `format` as a reserved keyword and reject the schema.
- When asked to open a “session” file, open the Pi session logs under `~/.openclaw/agents/<agentId>/sessions/*.jsonl` (use the `agent=<id>` value in the Runtime line of the system prompt; newest unless a specific ID is given), not the default `sessions.json`. If logs are needed from another machine, SSH via Tailscale and read the same path there.
- Do not rebuild the macOS app over SSH; rebuilds must be run directly on the Mac.
- Never send streaming/partial replies to external messaging surfaces (WhatsApp, Telegram); only final replies should be delivered there. Streaming/tool events may still go to internal UIs/control channel.
- Voice wake forwarding tips:
  - Command template should stay `openclaw-mac agent --message "${text}" --thinking low`; `VoiceWakeForwarder` already shell-escapes `${text}`. Don’t add extra quotes.
  - launchd PATH is minimal; ensure the app’s launch agent PATH includes standard system paths plus your pnpm bin (typically `$HOME/Library/pnpm`) so `pnpm`/`openclaw` binaries resolve when invoked via `openclaw-mac`.
- For manual `openclaw message send` messages that include `!`, use the heredoc pattern noted below to avoid the Bash tool’s escaping.
- Release guardrails: do not change version numbers without operator’s explicit consent; always ask permission before running any npm publish/release step.

## NPM + 1Password (publish/verify)

- Use the 1password skill; all `op` commands must run inside a fresh tmux session.
- Sign in: `eval "$(op signin --account my.1password.com)"` (app unlocked + integration on).
- OTP: `op read 'op://Private/Npmjs/one-time password?attribute=otp'`.
- Publish: `npm publish --access public --otp="<otp>"` (run from the package dir).
- Verify without local npmrc side effects: `npm view <pkg> version --userconfig "$(mktemp)"`.
- Kill the tmux session after publish.
