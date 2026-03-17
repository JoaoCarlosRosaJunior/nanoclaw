# Session Notes — 2026-03-17

## What Was Done

### 1. Initial Setup (`/setup`)

- **Git remotes**: Already configured — `origin` (user's fork), `upstream` (qwibitai/nanoclaw)
- **Bootstrap**: Node.js 22, dependencies, native modules — all passed
- **Environment**: Fresh install, no prior config
- **Container runtime**: Docker (already running on Linux)
- **Container image**: Built and tested successfully
- **Claude auth**: OAuth token via `claude setup-token` → `CLAUDE_CODE_OAUTH_TOKEN` in `.env`

### 2. Channel Setup

#### Telegram
- Added `telegram` remote, merged `nanoclaw-telegram` branch
- Resolved merge conflicts (package-lock.json, package.json, badge.svg)
- Installed `grammy` dependency, build clean, 50 tests passing
- Bot: `@junior_claw_assistant_bot`
- Configured `TELEGRAM_BOT_TOKEN` in `.env`

#### Slack
- Added `slack` remote, merged `nanoclaw-slack` branch
- Resolved merge conflicts (same files + `.env.example` — had to resolve manually due to `.env` security hook)
- After Slack merge, `grammy` was lost from `package.json` because `--theirs` overwrote it — manually re-added
- Installed `@slack/bolt`, build clean, 46 tests passing
- Configured `SLACK_BOT_TOKEN` and `SLACK_APP_TOKEN` in `.env`

#### Registrations
- Telegram DM: `tg:1163402205` → folder `telegram_main` (later changed to `telegram_personal`)
- Slack DM: `slack:D0ALZ4Q3WMU` → folder `slack_main`
- Telegram group: `tg:-5294239484` → folder `telegram_main` (MyAgents group)

### 3. Mount Allowlist & Service

- Empty mount allowlist (no external directory access)
- Systemd user service created, enabled, and started
- Verification: all checks passed (service running, credentials configured, channels authenticated, groups registered)

### 4. Agent Swarm (Telegram Bot Pool)

#### Problem
User wanted Agent Teams where each subagent appears as a different bot in Telegram.

#### Implementation
Modified 5 files:

1. **`src/config.ts`** — Added `TELEGRAM_BOT_POOL` config reading comma-separated tokens from `.env` via `readEnvFile`
2. **`src/channels/telegram.ts`** — Added `initBotPool()` and `sendPoolMessage()` functions. Pool bots are send-only `Api` instances (no polling). Sender-to-bot mapping is stable per group (round-robin assignment, `setMyName` on first use with 2s propagation delay)
3. **`src/ipc.ts`** — Updated IPC message routing: when `data.sender` exists and JID starts with `tg:`, routes through `sendPoolMessage` instead of `deps.sendMessage`
4. **`src/index.ts`** — Added pool initialization after channel creation loop
5. **`groups/telegram_main/CLAUDE.md`** — Added Agent Teams instructions (sender parameter usage, message formatting, team creation examples, lead agent behavior)

The `container/agent-runner/src/ipc-mcp-stdio.ts` already had the `sender` parameter on `send_message` — no changes needed there.

#### Issues Encountered
- **`TELEGRAM_BOT_POOL` not loading**: Config read from `process.env` only, but systemd doesn't load `.env`. Fixed by adding `TELEGRAM_BOT_POOL` to `readEnvFile` call
- **Telegram "channel" vs "group"**: User initially created a Telegram Channel (broadcast-only) instead of a Group. Bots can't receive messages in channels. Re-created as a proper Group
- **Main bot not responding to `/chatid` in group**: Group Privacy was enabled on main bot. Disabled via @BotFather + removed/re-added bot. Used mobile Telegram app since web client had issues adding bots
- **Auth 401 error**: OAuth token had expired. User refreshed via `claude setup-token`

#### Final state
- 3 pool bots initialized successfully
- Swarm working — lead agent spawns subagents that appear as separate bots in the Telegram group

### 5. Telegram Personal DM Re-registration

The original DM registration (`tg:1163402205` → `telegram_main`) was overwritten when the group was registered with the same folder. Re-registered DM as a separate entry:
- `tg:1163402205` → folder `telegram_personal` (main, no trigger required)

## Troubleshooting Notes

- **`.env` security hook**: User has a hook (`~/.claude/hooks/prevent-env-exfil.py`) that blocks any bash command touching `.env` files. All `.env` operations must be done manually by the user
- **Telegram Web limitations**: Adding bots to groups doesn't work reliably on web.telegram.org. Mobile app is more reliable
- **Slack "Sending messages to this app has been turned off"**: Fixed by enabling Messages Tab in App Home settings + checking "Allow users to send Slash commands and messages from the messages tab"

---

## Research: Multi-Agent Architecture in NanoClaw

### How NanoClaw Models Agents

Each "agent" is a **registered group** — a row in SQLite with its own:
- **Folder** (`groups/{name}/`) — persistent filesystem and memory
- **CLAUDE.md** — personality, role, domain instructions
- **Container** — isolated Docker environment per invocation
- **Session** — persisted Claude Code session for conversation continuity
- **Optional mounts** — host directories exposed via `containerConfig.additionalMounts`

Agents are created by registering groups via the main group's `register_group` MCP tool or the setup CLI.

### Key Capabilities for Specialized Agents

| Capability | How It Works |
|-----------|-------------|
| Per-agent personality | CLAUDE.md in the group's folder |
| Per-agent file access | `containerConfig.additionalMounts` |
| Per-agent memory | Files in `groups/{folder}/` persist across sessions |
| Scheduled tasks | `schedule_task` with cron/interval/once — great for daily quizzes, weekly reviews |
| Cross-agent coordination | Main group can schedule tasks for any group, read SQLite |
| Agent swarms | Sub-agents within a single container via Claude Code Teams |

### Proposed Agent Setup (Not Yet Implemented)

| Agent | Folder | Mounts | Features |
|-------|--------|--------|----------|
| Debug Engineer | `telegram_debug` | Logs dir (read-only) | Log analysis, error detection |
| Cloud Ops | `telegram_cloud` | `~/.aws` (read-only) | AWS CLI access, infrastructure context |
| Dev / PR Creator | `telegram_dev` | Project repos (read-write) | PR workflow, GitHub integration |
| Teacher | `telegram_teacher` | Study materials | Daily scheduled quizzes, answer evaluation |
| Performance Coach | `telegram_coach` | — | Daily logs, weekly assessment via scheduled task |
| Email Reader | `telegram_email` | — | Gmail integration via `/add-gmail` |
| Personal | `telegram_personal` | — | General assistant (already registered) |

### Approach Options

**Approach A — Separate Telegram groups per agent**: Create a Telegram group for each specialized agent. Same main bot, different registered group folders.

**Approach B — Separate Telegram bots per agent**: Create a distinct bot via @BotFather for each role. Each bot = one DM = one registered group. Cleanest separation — DM the Debug bot for debug, Cloud bot for AWS, etc. Requires NanoClaw code changes to support multiple bot tokens for polling (currently only one `TELEGRAM_BOT_TOKEN`).

### Limitations Discovered

- No per-group environment variables (all containers share same env)
- No per-group tool restrictions (same `allowedTools` list for all)
- No per-group model selection
- Concurrency limit is global (`MAX_CONCURRENT_CONTAINERS=5`)
- Agent swarm bot names are global (Telegram `setMyName` is not per-chat)
- No declarative multi-agent config file — groups registered one by one

### Sources

- NanoClaw GitHub: https://github.com/qwibitai/nanoclaw
- NanoClaw Official Site: https://nanoclaw.dev/
- The New Stack article on containerized agents
- VirtusLab blog on NanoClaw as personal AI butler
- BitDoze deploy guide
- Substack article on OpenClaw/NanoClaw patterns

---

## Research: Google Workspace CLI (`gws`) as Alternative to MCP Servers

### Context

User prefers CLI tools over MCP servers for agent integrations. Google recently published a CLI for all of Google Workspace.

### Google Workspace CLI (`gws`)

- **GitHub**: https://github.com/googleworkspace/cli
- **npm**: `@googleworkspace/cli`
- **Note**: Published under `googleworkspace` GitHub org but README states "This is not an officially supported Google product." Pre-v1.0, breaking changes expected.

### Installation

```bash
npm install -g @googleworkspace/cli
# or: brew install googleworkspace-cli
# or: cargo install --git https://github.com/googleworkspace/cli --locked
```

### Authentication

```bash
gws auth setup    # one-time: creates Cloud project, enables APIs, logs in
gws auth login    # subsequent logins
```

Supports `--readonly` flag — only requests `*.readonly` OAuth scopes, enforced at Google API level.

### Services Available

| Service | Key Commands | Agent Use Case |
|---------|-------------|----------------|
| **Gmail** | `+triage`, `+watch`, `messages list/get`, `threads list/get`, `labels list` | Email Reader agent — read-only inbox access |
| **Drive** | `files list/get/export`, `permissions list` | Access shared docs, specs, reports |
| **Calendar** | `events list/get/insert`, `calendars list` | Coach agent — see daily schedule, plan study sessions |
| **Sheets** | `spreadsheets get`, `values get/update` | Teacher agent — track study progress |
| **Docs** | `documents get/create` | Dev agent — read/write specs and docs |
| **Slides** | `presentations get/create` | Create presentations from data |
| **Chat** | `spaces list`, `messages list/create` | Google Chat integration |
| **Tasks** | `tasklists list`, `tasks list/insert` | Coach agent — to-do tracking |
| **Forms** | `forms get`, `responses list` | Read form/survey responses |
| **Admin** | `users list`, `groups list` | Workspace admin tasks |

### Why `gws` Over MCP for NanoClaw

1. **CLI-based** — agents call it via Bash, no MCP server process to manage
2. **Read-only enforcement at OAuth scope level** — not just prompt-based, Google API rejects write attempts
3. **Structured JSON output** — easy for agents to parse
4. **Dynamic API surface** — reads Google's Discovery Service at runtime, new endpoints auto-discovered
5. **Per-agent scoping** — each agent can auth with different scopes (email agent: `gmail.readonly`, coach: `calendar.readonly + tasks`)
6. **Single tool for everything** — replaces multiple MCP servers (Gmail, Drive, Calendar) with one CLI

### Integration Plan for NanoClaw

For each specialized agent:
1. Install `gws` inside the agent container (add to Dockerfile)
2. Auth with appropriate `--readonly` or scoped permissions
3. Mount `~/.gws` credentials into the container
4. Agent uses `gws <service> <command>` via Bash — no MCP server needed

Example per-agent scopes:
- **Email Reader**: `gmail.readonly`
- **Performance Coach**: `calendar.readonly`, `tasks`
- **Dev / PR Creator**: `drive.readonly`, `docs.readonly`
- **Teacher**: `sheets`, `calendar.readonly`

### Alternative: gogcli (`gog`)

Community-built Go CLI by Peter Steinberger (https://github.com/steipete/gogcli). More mature, multi-account support, OS keyring credential storage. Also supports `--gmail-scope readonly`. Worth considering as a fallback if `gws` has stability issues (it's pre-v1.0).
