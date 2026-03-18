# Known Limitations

Last updated: 2026-03-18

## Agent Routing & Communication

1. **No multi-agent routing in a single group** — Can't `@Engineering` and `@Finance` in the same Telegram group and route to different agents. Each group is bound to one agent template.

2. **No inter-agent communication** — Agents can't talk to each other at the NanoClaw level. Sub-agents are Claude Code's internal Teams feature (ephemeral, within one container), not persistent NanoClaw agents.

3. **DM is limited to one agent** — Can't have multiple DM conversations with different agents using the same bot. The DM is always the main/general agent. Specialized agents require separate groups or channels.

4. **No cross-group context** — Agents in different groups can't share information. Main group can read SQLite but can't access other groups' files or memory.

5. **Conversation history is not searchable across groups** — Each group's `conversations/` folder is isolated.

## Permissions & Security

6. **No approval/decline for agent actions** — `allowDangerouslySkipPermissions: true`, everything executes immediately. No interactive permission flow via messaging. The deny list blocks specific commands, but there's no "ask before running" mode.

7. **`is_from_me` is broken for Telegram and Slack** — Hardcoded `false` for all Telegram users. Means "bot sent this" for Slack. Session commands (`/compact`) only work in main groups because of this. Non-main group authorization via `is_from_me` is non-functional for these channels.

8. **OAuth token expiration** — `CLAUDE_CODE_OAUTH_TOKEN` expires periodically. No auto-refresh, no user notification. Agent fails with 401 until the token is manually refreshed via `claude setup-token`.

## User Experience

9. **No status/progress visibility** — User has no way to know if the agent is working, stuck, or errored without checking logs. The only feedback is the eventual response or silence.

10. **Telegram typing indicator disappears after 5 seconds** — Only sent once at container spawn, not refreshed during long tasks. For tasks taking minutes, the user sees no activity.

11. **Slack has no typing indicator** — Slack Bot API doesn't support typing indicators for bots. This is a platform limitation.

12. **Slack `/commands` intercepted by Slack** — `/compact` and other slash commands don't work in Slack because Slack treats anything starting with `/` as a Slack command. Workaround: use a leading space (` /compact`).

13. **No message delivery confirmation** — User doesn't know if their message was received and queued vs lost.

14. **Context window management is opaque** — User doesn't know when the agent is near context limit or when compaction happened automatically.

15. **No model selection command** — Model changes require the agent to edit `settings.json` (via CLAUDE.md instruction) and take effect on next container spawn, not immediately.

## Infrastructure

16. **All groups share the same Docker image, skills, and environment** — No per-group customization of tools or credentials. The agent template system (task 003) will fix this.

17. **Stale agent-runner cache after MCP tool changes** — When new MCP tools are added or modified, per-group copies in `data/sessions/*/agent-runner-src/` must be manually deleted (`rm -rf data/sessions/*/agent-runner-src`) or the changes won't take effect. Produces confusing "tool not available" errors.

18. **Container idle timeout is 30 seconds** — After the last result, the container waits 30 seconds for follow-up messages before exiting. Configurable via `IDLE_TIMEOUT` env var but applies to all groups uniformly.

19. **`MAX_CONCURRENT_CONTAINERS` defaults to 5** — A chatty group can starve others. The limit is global, not per-group.

20. **Some config values don't load from `.env` under systemd** — Only values explicitly added to `readEnvFile()` in `config.ts` are read from the `.env` file. Others only work via `process.env` (which systemd doesn't set). Fixed for `TELEGRAM_BOT_POOL`, `MAX_CONCURRENT_CONTAINERS`, and `ATTACHMENT_RETENTION_DAYS`. Others like `CONTAINER_TIMEOUT`, `IDLE_TIMEOUT` still read from `process.env` only.

21. **No graceful degradation when Docker is down** — If Docker daemon stops, NanoClaw logs errors but doesn't notify the user via messaging.

22. **No rate limiting** — If someone floods a group with messages, every message gets processed. Could exhaust all container slots.

23. **Skill marketplace is not live** — `docs/skills-as-branches.md` describes a future plugin marketplace architecture, but `claude plugin install` doesn't work yet. Skills are currently installed via git branch merges or manual copying.

## File Transfer

24. **Telegram bot file download limit is 20MB** — Telegram Bot API limits `getFile` to 20MB. Larger files sent by users can't be downloaded by the bot.

25. **Outbound file size limit is 50MB** — Files larger than 50MB can't be sent through channels. The agent sends a text message with the server path instead.

26. **No file type validation on inbound** — Any file type is accepted and downloaded. No malware scanning or content validation.

## Agent Swarm (Telegram)

27. **Agent swarm bot names are global** — Telegram `setMyName` changes the bot name globally, not per-chat. Two groups using the same pool bot with different sender names will conflict.

28. **Swarm sub-agents are ephemeral** — Sub-agents spawned via Claude Code Teams exist only for the duration of the lead agent's session. They don't persist independently.

## Logs & Debugging

29. **Logs are difficult to read** — `nanoclaw.log` mixes all groups, channels, and subsystems into one file. No structured log viewer, no per-group filtering, no log levels in the default view.

30. **Container logs require manual inspection** — Container-level logs are in `groups/{folder}/logs/container-*.log`. No aggregation, no automatic error surfacing. To see what the agent is doing in real-time: `docker logs -f $(docker ps --filter "name=nanoclaw" -q)`.

## Operational Reference

### Common commands

```bash
# Service
systemctl --user status nanoclaw
systemctl --user restart nanoclaw

# Logs
tail -f logs/nanoclaw.log
docker logs -f $(docker ps --filter "name=nanoclaw" -q)

# After code changes
npm run build && systemctl --user restart nanoclaw

# After Dockerfile or MCP tool changes
./container/build.sh
rm -rf data/sessions/*/agent-runner-src
systemctl --user restart nanoclaw

# Check registered groups
sqlite3 store/messages.db "SELECT jid, name, folder FROM registered_groups"

# Regenerate settings.json for a group (picks up new defaults)
rm data/sessions/{folder}/.claude/settings.json

# Container status
docker ps --filter "name=nanoclaw"
```

### How secrets are loaded

| Secret | Loaded by | File |
|--------|-----------|------|
| `CLAUDE_CODE_OAUTH_TOKEN` | `readEnvFile()` in `credential-proxy.ts` | `.env` |
| `ANTHROPIC_API_KEY` | `readEnvFile()` in `credential-proxy.ts` | `.env` |
| `SLACK_BOT_TOKEN` | `readEnvFile()` in `slack.ts` | `.env` |
| `SLACK_APP_TOKEN` | `readEnvFile()` in `slack.ts` | `.env` |
| `TELEGRAM_BOT_TOKEN` | `readEnvFile()` in `telegram.ts` | `.env` |
| `TELEGRAM_BOT_POOL` | `readEnvFile()` in `config.ts` | `.env` |

### How config values are loaded

| Config | Loaded from `.env`? | Default |
|--------|---------------------|---------|
| `ASSISTANT_NAME` | Yes | `Andy` |
| `TELEGRAM_BOT_POOL` | Yes | empty |
| `MAX_CONCURRENT_CONTAINERS` | Yes | `5` |
| `ATTACHMENT_RETENTION_DAYS` | Yes | `7` |
| `CONTAINER_TIMEOUT` | No (`process.env` only) | `1800000` (30 min) |
| `IDLE_TIMEOUT` | No (`process.env` only) | `1800000` (30 min) |
| `CONTAINER_IMAGE` | No (`process.env` only) | `nanoclaw-agent:latest` |
| `CREDENTIAL_PROXY_PORT` | No (`process.env` only) | `3001` |
| `CONTAINER_MAX_OUTPUT_SIZE` | No (`process.env` only) | `10485760` (10MB) |
