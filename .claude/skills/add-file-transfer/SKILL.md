---
name: add-file-transfer
description: Add file send/receive support for Slack and Telegram channels. Agents can send files via send_file MCP tool. Users can send files that get downloaded and passed to the agent. Includes auto-cleanup of inbound files after 7 days.
---

# Add File Transfer Support

Adds bidirectional file transfer to NanoClaw channels (Slack and Telegram). Agents can send files to users, and users can send files to agents.

## Phase 1: Pre-flight

### Check if already applied

```bash
grep -q "send_file" container/agent-runner/src/ipc-mcp-stdio.ts && echo "Already applied" || echo "Not applied"
```

If already applied, skip to Phase 3 (Verify).

### Prerequisites

- Slack and/or Telegram channels must be set up first (`/add-slack`, `/add-telegram`)
- For Slack file downloads: the bot needs `files:read` and `files:write` OAuth scopes. If not present, guide the user to add them at [api.slack.com/apps](https://api.slack.com/apps) → OAuth & Permissions → Bot Token Scopes. **The app must be reinstalled after adding scopes**, and the new bot token must be updated in `.env`.

## Phase 2: Apply Code Changes

### 2a. Add `sendFile()` to Channel interface

**File:** `src/types.ts`

Add to the `Channel` interface, before `setTyping`:

```typescript
sendFile?(jid: string, filePath: string, caption?: string): Promise<void>;
```

### 2b. Implement `sendFile()` in Telegram channel

**File:** `src/channels/telegram.ts`

1. Add imports at top: `fs`, `path`, `InputFile` (from grammy), `GROUPS_DIR` (from config)

2. Add `sendFile()` method to `TelegramChannel` class (before `setTyping`):
   - Extract numeric chat ID from JID
   - Create `InputFile` from the file path
   - Auto-detect images by extension (`.jpg`, `.jpeg`, `.png`, `.gif`, `.webp`) → use `sendPhoto()`
   - All other files → use `sendDocument()`
   - Include caption if provided

3. Update inbound non-text message handlers to download files:
   - Add `downloadFile()` helper that calls `ctx.api.getFile()`, downloads from Telegram API, saves to `groups/{folder}/user_attachments/{timestamp}/{filename}`
   - Before saving: cleanup `user_attachments/` folders older than 7 days
   - Update `storeNonText()` to be async, accept `fileId` and `filename`, call `downloadFile()`, include path in message content as `[File: /workspace/group/user_attachments/{ts}/{filename}]`
   - **IMPORTANT**: All media handlers (`message:photo`, `message:video`, `message:voice`, `message:audio`, `message:document`) must be `async` and `await` the `storeNonText()` call. Without `await`, the download runs detached and causes multi-minute delays.
   - For `message:photo`: use the largest photo (last element in `ctx.message.photo` array)

### 2c. Implement `sendFile()` in Slack channel

**File:** `src/channels/slack.ts`

1. Add imports at top: `fs`, `path`, `GROUPS_DIR` (from config)

2. Add `sendFile()` method to `SlackChannel` class (before `isConnected`):
   - Use `this.app.client.filesUploadV2({ channel_id, file: fs.createReadStream(filePath), filename, initial_comment: caption })`

3. Update inbound message handler to download files:
   - Allow `file_share` subtype in the subtype filter
   - Allow messages with files but no text (update the `!msg.text` guard)
   - After resolving sender name, check `msg.files` array
   - For each file: download using `url_private_download` with bot token auth
   - **IMPORTANT**: Slack redirects file downloads and strips the Authorization header. Use `redirect: 'manual'`, check for 3xx status, then re-fetch the redirect URL with the token re-attached:
     ```typescript
     let resp = await fetch(url, { headers: { Authorization: `Bearer ${token}` }, redirect: 'manual' });
     if (resp.status >= 300 && resp.status < 400 && resp.headers.get('location')) {
       resp = await fetch(resp.headers.get('location')!, { headers: { Authorization: `Bearer ${token}` } });
     }
     ```
   - If response content-type is `text/html`, log a warning — this means auth failed (likely missing `files:read` scope)
   - Save to `groups/{folder}/user_attachments/{timestamp}/{filename}`
   - Before saving: cleanup `user_attachments/` folders older than 7 days
   - Prepend to message content: `[File: /workspace/group/user_attachments/{ts}/{filename}]`

### 2d. Add `send_file` MCP tool

**File:** `container/agent-runner/src/ipc-mcp-stdio.ts`

Add a new tool after `send_message`:

```typescript
server.tool(
  'send_file',
  'Send a file to the user or group. Use for sharing generated files, reports, images, etc.',
  {
    path: z.string().describe('Absolute path to the file (e.g., /workspace/group/agent_attachments/reports/report.csv)'),
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

### 2e. Handle file IPC messages on the host

**File:** `src/ipc.ts`

1. Add `GROUPS_DIR` import from config
2. Add `MAX_FILE_SIZE` constant: `50 * 1024 * 1024` (50MB)
3. Add `sendFile` to `IpcDeps` interface
4. In `processIpcFiles`, add handling for `data.type === 'file'`:
   - Translate container path to host path: `/workspace/group/X` → `groups/{folder}/X`
   - Check file exists
   - Check file size < 50MB. If too large: send text message with server path instead
   - Call `deps.sendFile(chatJid, hostPath, caption)`
   - Same authorization check as messages (isMain or matching group folder)

### 2f. Wire `sendFile` in orchestrator

**File:** `src/index.ts`

In the `startIpcWatcher()` deps, add `sendFile`:

```typescript
sendFile: (jid, filePath, caption) => {
  const channel = findChannel(channels, jid);
  if (!channel) throw new Error(`No channel for JID: ${jid}`);
  if (!channel.sendFile) {
    return channel.sendMessage(jid, `File available at: ${filePath}`);
  }
  return channel.sendFile(jid, filePath, caption);
},
```

### 2g. Update tests

- `src/ipc-auth.test.ts`: Add `sendFile: async () => {}` to the mock `IpcDeps`
- `src/channels/telegram.test.ts`: Update `createMediaCtx` to include `photo` array, `video`, `voice`, `audio`, `document` objects, and `api.getFile` mock

### 2h. Update CLAUDE.md

**File:** `groups/global/CLAUDE.md`

Add sections for Sending Files and Receiving Files. Key points:
- Agent saves files to `/workspace/group/agent_attachments/` (organized in subdirectories)
- Agent sends via `mcp__nanoclaw__send_file({ path, caption })`
- User files arrive as `[File: /workspace/group/user_attachments/{ts}/{filename}]` in message content
- When editing received files, save to `agent_attachments/`, not back to `user_attachments/`
- `user_attachments/` auto-cleaned after 7 days
- Max send size: 50MB

### 2i. Clear stale agent-runner copies

```bash
rm -rf data/sessions/*/agent-runner-src
```

Required because per-group agent-runner copies are cached and won't pick up the new `send_file` tool otherwise.

## Phase 3: Validate

```bash
npm run build
npm test
./container/build.sh
```

All tests must pass and container must build cleanly.

## Phase 4: Restart

```bash
systemctl --user restart nanoclaw   # Linux
# macOS: launchctl kickstart -k gui/$(id -u)/com.nanoclaw
```

## Phase 5: Verify

### Test inbound (user → agent)

1. **Slack**: Send a file (image, PDF, Excel) in the bot's DM. Agent should see `[File: /workspace/group/user_attachments/...]` and be able to read it.
2. **Telegram**: Send a document or photo. Same behavior.
3. Verify files are saved: `ls groups/{folder}/user_attachments/`

### Test outbound (agent → user)

1. Ask the agent: "Create a test CSV file and send it to me"
2. Agent should create the file in `agent_attachments/`, then call `send_file`
3. File should appear in your chat (image inline on Telegram, download link on Slack)

### Test file size limit

1. Ask the agent to create a large file and send it
2. Files > 50MB should result in a text message with the server path

### Test cleanup

1. Verify old `user_attachments/` folders are cleaned when new files arrive
2. `agent_attachments/` should NOT be cleaned

## Troubleshooting

### Agent says "send_file tool isn't available"

Stale agent-runner copies. Fix:
```bash
rm -rf data/sessions/*/agent-runner-src
systemctl --user restart nanoclaw
```

### Slack file download returns HTML instead of file

Missing `files:read` OAuth scope. Fix:
1. Go to [api.slack.com/apps](https://api.slack.com/apps) → your app → OAuth & Permissions
2. Add `files:read` and `files:write` to Bot Token Scopes
3. **Reinstall the app** (scope changes require reinstall)
4. Update the new bot token in `.env`
5. Restart NanoClaw

### Telegram file download slow or delayed

Ensure media handlers use `async`/`await`:
```typescript
this.bot.on('message:document', async (ctx) => {
  await storeNonText(ctx, ...);  // MUST await
});
```
Without `await`, downloads run detached and cause multi-minute delays.

### File not found on send

The IPC handler translates `/workspace/group/X` to `groups/{folder}/X`. If the agent writes to a path outside `/workspace/group/`, the translation fails. Agent should always save files under `/workspace/group/agent_attachments/`.

## File Structure

```
groups/{group_name}/
├── CLAUDE.md                    ← agent personality + instructions
├── conversations/               ← compacted transcripts
├── logs/                        ← container logs
├── user_attachments/            ← inbound files (auto-cleaned 7 days)
│   └── {timestamp}/
│       └── uploaded_file.xlsx
└── agent_attachments/           ← agent workspace (organized by agent)
    ├── reports/
    ├── data/
    └── research/
```
