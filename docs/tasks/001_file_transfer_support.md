# 001 — File Transfer Support (Send & Receive Files via Channels)

## Problem

NanoClaw's entire pipeline is text-only. Agents can create files on their workspace (`/workspace/group/`) but cannot send them to the user through messaging channels. Similarly, files sent by users (images, PDFs, documents) are silently dropped by all channels.

## What We Want

1. **Outbound**: Agent creates a file → calls `send_file` MCP tool → host reads the file from the group folder → sends it via the channel's platform API
2. **Inbound**: User sends a file → channel downloads it to the group's attachments folder → message content includes the file path → agent can read/process it

## Decisions

| Decision | Choice | Reason |
|----------|--------|--------|
| Outbound trigger | New `send_file` MCP tool (not extending `send_message`) | Single responsibility, cleaner API, no risk of breaking existing send_message |
| Inbound format | Append file path to message content string | Simpler, fewer core file changes, matches existing pattern (`[Photo]`, `[Document: name]`) |
| Inbound storage | `groups/{folder}/attachments/{timestamp}/` | Timestamp folders enable easy age-based cleanup |
| File types | All file types supported | Images auto-detect for inline rendering, everything else as document |
| Max file size | 50MB (Telegram bot limit) | Files exceeding limit: agent sends text message with server path instead |
| Telegram rendering | Auto-detect: images → `sendPhoto()` (inline preview), other → `sendDocument()` (download link) | Better UX — users see images directly in chat |
| Cleanup | On-download cleanup of folders older than 7 days | No cron needed — piggybacks on new attachment arrival |
| Scope | Slack and Telegram only | The two currently installed channels |

## Architecture

### Outbound Flow

```
Agent creates file at /workspace/group/report.csv
    ↓
Agent calls mcp__nanoclaw__send_file({ path: "/workspace/group/report.csv", caption: "Here's the report" })
    ↓
IPC file written: { type: "file", chatJid: "...", path: "/workspace/group/report.csv", caption: "..." }
    ↓
Host IPC watcher reads it
    ↓
Path translation: /workspace/group/report.csv → groups/{folder}/report.csv (host path)
    ↓
File size check: if > 50MB → send text message with server path instead
    ↓
channel.sendFile(jid, hostPath, caption)
    ↓
Slack: app.client.files.uploadV2({ channel, file, filename, initial_comment })
Telegram: bot.api.sendDocument(chatId, file) or bot.api.sendPhoto(chatId, file) for images
```

### Inbound Flow

```
User sends a file in Slack/Telegram
    ↓
Channel event handler detects file attachment
    ↓
Cleanup: delete attachment folders older than 7 days
    ↓
Download file to groups/{folder}/attachments/{timestamp}/{original_filename}
    ↓
Store message with content: "[File: /workspace/group/attachments/{timestamp}/{filename}] optional caption"
    ↓
Agent receives in prompt:
<message sender="João" time="17:30">[File: /workspace/group/attachments/1710000000/report.pdf] Check this report</message>
    ↓
Agent can Read the file, process it, etc.
```

### Slack File Download

Slack files require auth to download. Use `app.client.files.info()` to get the download URL, then fetch with the bot token in the Authorization header.

### Telegram File Download

Grammy provides `ctx.getFile()` which returns a file path on Telegram's servers. Download via `https://api.telegram.org/file/bot{token}/{file_path}`.

## Files to Read

Before implementing, read these files in order:

### Core architecture
- `src/types.ts` — `Channel` interface (line 82-93), `NewMessage` type (line 45-54). Add `sendFile?()` to Channel
- `src/router.ts` — Outbound message routing (~50 lines). Add `routeFile()` or extend existing routing
- `src/ipc.ts` — IPC message processing (lines 67-108). Add `type: "file"` handling alongside existing `type: "message"`

### IPC / MCP tool
- `container/agent-runner/src/ipc-mcp-stdio.ts` — `send_message` tool (lines 42-63). Create `send_file` following same pattern

### Channel implementations
- `src/channels/slack.ts` — `sendMessage()` (lines 159-191). Add `sendFile()` using `app.client.files.uploadV2()`
- `src/channels/telegram.ts` — `sendMessage()` (lines 240-266). Add `sendFile()` using `bot.api.sendDocument()` / `sendPhoto()`
- `src/channels/telegram.ts` — Non-text handlers (lines 202-215). Reference for inbound pattern: `[Photo]`, `[Document: name]`, `[Voice message]`

### Container mounts
- `src/container-runner.ts` — Mount mapping (lines 60-176). `/workspace/group` → `groups/{folder}/` on host. Path translation needed for outbound files

### Existing file handling (reference)
- `.claude/skills/add-image-vision/SKILL.md` — WhatsApp image attachments pattern
- `.claude/skills/add-pdf-reader/SKILL.md` — WhatsApp PDF attachments pattern

## Implementation Steps

### Step 1: Add `sendFile()` to Channel interface

**File:** `src/types.ts`

```typescript
export interface Channel {
  // ... existing methods ...
  sendFile?(jid: string, filePath: string, caption?: string): Promise<void>;
}
```

### Step 2: Implement `sendFile()` in Telegram channel

**File:** `src/channels/telegram.ts`

- Auto-detect image types by extension (`.jpg`, `.jpeg`, `.png`, `.gif`, `.webp`) → use `sendPhoto()`
- Everything else → use `sendDocument()`
- File size check: if > 50MB, throw or return false
- Include caption as `{ caption }` option

### Step 3: Implement `sendFile()` in Slack channel

**File:** `src/channels/slack.ts`

- Use `app.client.files.uploadV2({ channel_id, file: fs.createReadStream(path), filename, initial_comment: caption })`
- File size check: if > 50MB, throw or return false

### Step 4: Add `send_file` MCP tool

**File:** `container/agent-runner/src/ipc-mcp-stdio.ts`

```typescript
server.tool(
  'send_file',
  'Send a file to the user or group. Use for sharing generated files, reports, images, etc.',
  {
    path: z.string().describe('Absolute path to the file inside the container (e.g., /workspace/group/report.csv)'),
    caption: z.string().optional().describe('Optional caption or description for the file'),
  },
  async (args) => {
    const data = {
      type: 'file',
      chatJid,
      path: args.path,
      caption: args.caption || undefined,
      groupFolder,
      timestamp: new Date().toISOString(),
    };
    writeIpcFile(MESSAGES_DIR, data);
    return { content: [{ type: 'text' as const, text: 'File sent.' }] };
  },
);
```

### Step 5: Handle file IPC messages on the host

**File:** `src/ipc.ts`

Add a new branch for `data.type === 'file'` alongside the existing `data.type === 'message'`:

- Translate container path to host path: `/workspace/group/X` → `groups/{folder}/X`
- Check file exists on host
- Check file size (< 50MB)
- If too large: call `deps.sendMessage(chatJid, "File exceeds 50MB limit. Available on server at: groups/{folder}/X")`
- If OK: call `channel.sendFile(chatJid, hostPath, data.caption)`
- Authorization: same check as messages (isMain or matching group folder)

### Step 6: Inbound — Telegram file downloads

**File:** `src/channels/telegram.ts`

Update the existing non-text handlers (`message:photo`, `message:document`, `message:video`, etc.):

- Call `ctx.getFile()` to get file info
- Download from `https://api.telegram.org/file/bot{token}/{file_path}`
- Save to `groups/{folder}/attachments/{timestamp}/{original_filename}`
- Before saving: cleanup folders older than 7 days
- Update message content to include the path: `[File: /workspace/group/attachments/{timestamp}/{filename}] caption`

### Step 7: Inbound — Slack file downloads

**File:** `src/channels/slack.ts`

Update the message event handler:

- Check `msg.files` array on incoming messages
- For each file: download using `url_private_download` with bot token auth
- Save to `groups/{folder}/attachments/{timestamp}/{original_filename}`
- Before saving: cleanup folders older than 7 days
- Prepend to message content: `[File: /workspace/group/attachments/{timestamp}/{filename}] msg.text`

### Step 8: Cleanup utility

**File:** `src/channels/telegram.ts` and `src/channels/slack.ts` (or shared utility)

```typescript
function cleanupOldAttachments(attachmentsDir: string, maxAgeDays: number = 7): void {
  if (!fs.existsSync(attachmentsDir)) return;
  const cutoff = Date.now() - maxAgeDays * 24 * 60 * 60 * 1000;
  for (const dir of fs.readdirSync(attachmentsDir)) {
    if (parseInt(dir) < cutoff) {
      fs.rmSync(path.join(attachmentsDir, dir), { recursive: true });
    }
  }
}
```

### Step 9: Update CLAUDE.md

**File:** `groups/global/CLAUDE.md`

Add section explaining file capabilities:

```markdown
## Sending Files

Use `mcp__nanoclaw__send_file` to send files to the user:
- `send_file({ path: "/workspace/group/report.csv", caption: "Monthly report" })`
- Supported: any file type (images show inline, others as download)
- Max size: 50MB. Larger files stay on the server.

## Receiving Files

When users send files, you'll see them in the message as:
[File: /workspace/group/attachments/{timestamp}/filename.ext] optional caption

Use the Read tool to view the file contents. When editing a received file, always save your work to /workspace/group/ — not back to the attachments folder. Attachments are temporary and cleaned up after 7 days.
```

### Step 10: Tests

Add tests for:
- `send_file` MCP tool writes correct IPC JSON
- IPC handler translates container paths to host paths
- IPC handler rejects files > 50MB with text fallback
- IPC handler blocks unauthorized file sends (same auth as messages)
- Telegram `sendFile()` uses `sendPhoto` for images, `sendDocument` for others
- Slack `sendFile()` calls `files.uploadV2`
- Cleanup deletes folders older than 7 days
- Cleanup preserves recent folders
- Inbound: file download + content string format

## Files Changed Summary

| File | Change | Core file? |
|------|--------|------------|
| `src/types.ts` | Add `sendFile?()` to Channel interface | Yes |
| `src/ipc.ts` | Add `type: "file"` handling with path translation | Yes |
| `src/channels/telegram.ts` | Add `sendFile()`, update inbound handlers to download files | Channel (lower risk) |
| `src/channels/slack.ts` | Add `sendFile()`, update inbound handler to download files | Channel (lower risk) |
| `container/agent-runner/src/ipc-mcp-stdio.ts` | Add `send_file` MCP tool | Yes |
| `groups/global/CLAUDE.md` | Add file send/receive documentation | No |
| `groups/main/CLAUDE.md` | Reference global docs | No |
