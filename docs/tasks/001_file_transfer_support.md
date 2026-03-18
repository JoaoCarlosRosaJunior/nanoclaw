# 001 — File Transfer Support (Send & Receive Files via Channels)

## Problem

NanoClaw's entire pipeline is text-only. Agents can create files on their workspace (`/workspace/group/`) but cannot send them to the user through messaging channels. Similarly, files sent by users (images, PDFs, documents) are silently dropped by all channels.

## What We Want

1. **Outbound**: Agent creates a file → sends it to the user via Slack/Telegram (each channel uses its own platform API)
2. **Inbound**: User sends a file → channel downloads it to the group folder → agent receives the file path in its prompt

## Files to Read

Before starting, the next agent should read and understand these files in order:

### Core architecture (understand the data flow)
- `src/types.ts` — The `Channel` interface (line 82-93) and `NewMessage` type (line 45-54). This is where `sendFile()` would be added
- `src/router.ts` — How outbound messages are routed to channels. Simple, ~50 lines
- `src/ipc.ts` — How IPC messages from the container are processed (lines 67-108). The `data.type === 'message'` branch is where file routing would go

### IPC / MCP tool (how the agent triggers actions)
- `container/agent-runner/src/ipc-mcp-stdio.ts` — The `send_message` tool definition (lines 42-63). A `send_file` tool would follow the same pattern

### Channel implementations (platform-specific sending)
- `src/channels/slack.ts` — `sendMessage()` method (lines 159-191). Uses `app.client.chat.postMessage()`. File upload would use `app.client.files.uploadV2()`
- `src/channels/telegram.ts` — `sendMessage()` method (lines 240-266). Uses `bot.api.sendMessage()`. File upload would use `bot.api.sendDocument()` / `sendPhoto()`

### Existing file handling skills (reference for inbound pattern)
- `.claude/skills/add-image-vision/SKILL.md` — How WhatsApp image attachments are handled (download → mount → pass path to agent)
- `.claude/skills/add-pdf-reader/SKILL.md` — Same pattern for PDFs

### Container mounts (understand where files live)
- `src/container-runner.ts` — Lines 60-176. Mount mapping: `/workspace/group` → `groups/{folder}/` on host. Files the agent creates at `/workspace/group/` are accessible on the host at `groups/{folder}/`

## Scope

- Slack and Telegram channels only (the two currently installed)
- Outbound: agent → user (send files created in `/workspace/group/`)
- Inbound: user → agent (download to `groups/{folder}/inbox/`, pass path in prompt)
- No WhatsApp or Discord (not installed)
