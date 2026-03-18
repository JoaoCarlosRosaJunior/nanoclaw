# 002 — Google Workspace CLI (`gws`) Integration

## Goal

Give NanoClaw agents access to Google Workspace (Gmail, Drive, Docs, Sheets, Slides, Calendar) via the `gws` CLI tool. Agents use it via Bash — no MCP server needed. Security enforced at three layers: OAuth scopes, deny list, and CLAUDE.md instructions.

## Current State

- `gws` is installed and authenticated on the host machine
- Credentials at `~/.config/gws/` (encrypted with AES-256-GCM, key in `.encryption_key`)
- OAuth scopes: gmail.readonly, drive.readonly, drive.file, drive.appdata, documents, spreadsheets, presentations, calendar.readonly, calendar.events.readonly, calendar.freebusy, calendar.settings.readonly
- Deny list already defined (see below)
- `gws` is NOT installed in the container image
- `gws` credentials are NOT mounted into containers

## What the Agent Can Do (by scope)

| Service | Read | Create/Write | Delete/Modify |
|---------|------|-------------|---------------|
| Gmail | Read emails, triage inbox | No | No |
| Calendar | View events, agenda, freebusy | No | No |
| Drive | List, search, download files | Upload, create folders | No |
| Docs | Read documents | Create, write, append | No |
| Sheets | Read spreadsheets | Create, write, append rows | No |
| Slides | Read presentations | Create, update slides | No |

## Implementation Steps

### Step 1: Install `gws` in the Docker image

**File:** `container/Dockerfile`

Add `gws` installation. The CLI is distributed via npm:

```dockerfile
RUN npm install -g @googleworkspace/cli
```

After editing, rebuild: `./container/build.sh`

### Step 2: Mount credentials into containers

**File:** `src/container-runner.ts`

In the `getVolumeMounts()` function (around line 60), add a mount for `~/.config/gws/`:

```typescript
// Mount gws credentials (read-only — agents can't modify auth)
const gwsConfigDir = path.join(os.homedir(), '.config', 'gws');
if (fs.existsSync(gwsConfigDir)) {
  mounts.push({
    hostPath: gwsConfigDir,
    containerPath: '/home/node/.config/gws',
    readonly: true,
  });
}
```

### Step 3: Set environment variables for `gws` in the container

**File:** `container/agent-runner/src/index.ts`

In the `sdkEnv` object (find where env vars are set for the SDK), add:

```typescript
GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND: 'file',
```

This tells `gws` to read the encryption key from the `.encryption_key` file instead of trying to use the OS keyring (Docker containers don't have one).

Alternatively, set this in the Dockerfile:

```dockerfile
ENV GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file
```

### Step 4: Copy agent skills to `container/skills/`

Copy these skill folders from `/home/nuviz/Documents/projects/external-repos/gws-cli/skills/` to `container/skills/`:

**Core service skills:**
- `gws-shared` — Required. Auth patterns, CLI syntax, flags, security rules
- `gws-gmail` — Full Gmail API reference (read, triage, all methods)
- `gws-gmail-triage` — Unread inbox summary helper
- `gws-drive` — Drive API (upload, organize, permissions)
- `gws-docs` — Docs API (write, create, get)
- `gws-sheets` — Sheets API (read, append, all methods)
- `gws-slides` — Slides API (create, get, batchUpdate)
- `gws-calendar` — Calendar API (agenda, freebusy, events)

**Workflow skills:**
- `gws-workflow` — Includes standup, meeting-prep, weekly-digest, email-to-task

Total: 9 skill folders.

These get auto-synced to all groups on container spawn (via `container-runner.ts` lines 149-159).

### Step 5: Add deny list to default settings.json

**File:** `src/container-runner.ts`

Update the default `settings.json` template (lines 129-142) to include the deny list:

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1",
    "CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD": "1",
    "CLAUDE_CODE_DISABLE_AUTO_MEMORY": "0"
  },
  "permissions": {
    "deny": [
      "Bash(gws gmail * send *)",
      "Bash(gws gmail * delete *)",
      "Bash(gws gmail * trash *)",
      "Bash(gws gmail * modify *)",
      "Bash(gws gmail * insert *)",
      "Bash(gws drive * delete *)",
      "Bash(gws drive * trash *)",
      "Bash(gws calendar * delete *)",
      "Bash(gws calendar * insert *)",
      "Bash(gws calendar * update *)",
      "Bash(gws calendar * patch *)",
      "Bash(gws docs * delete *)",
      "Bash(gws sheets * delete *)",
      "Bash(gws slides * delete *)"
    ]
  }
}
```

**Important:** This only applies to NEW groups. Existing groups (`slack_main`, `telegram_main`, `telegram_personal`) need their `data/sessions/{folder}/.claude/settings.json` updated manually (or deleted to regenerate).

### Step 6: Update CLAUDE.md files

#### `groups/global/CLAUDE.md` — Add after the existing content:

```markdown
## Google Workspace Access

You have access to Google Workspace via the `gws` CLI tool. Run commands via Bash.

### Permissions

You have READ-ONLY access to:
- *Gmail*: Read and search emails, triage inbox (`gws gmail +triage`, `gws gmail +read`)
- *Calendar*: View events, agenda, free/busy (`gws calendar +agenda`)

You have READ and WRITE access to:
- *Google Drive*: List, search, download, upload files, create folders
- *Google Docs*: Read, create, and write documents (`gws docs +write`)
- *Google Sheets*: Read, create, write, append rows (`gws sheets +read`, `gws sheets +append`)
- *Google Slides*: Read, create, update presentations

You DO NOT have permission to:
- Send, delete, or modify emails
- Create, delete, or modify calendar events
- Delete files from Drive, Docs, Sheets, or Slides

These restrictions are enforced at the OAuth scope level — commands will fail if attempted.

### Quick Reference

```bash
# Gmail
gws gmail +triage                              # Inbox summary
gws gmail +read --message-id MSG_ID            # Read a message

# Calendar
gws calendar +agenda                           # Today's events
gws calendar +agenda --days 7                  # Next 7 days

# Drive
gws drive files list --query "name contains 'report'"
gws drive files export --file-id ID --mime-type text/plain

# Docs
gws docs +write --document-id DOC_ID --text "content"
gws docs documents create --title "New Doc"

# Sheets
gws sheets +read --spreadsheet-id ID --range "Sheet1!A1:D10"
gws sheets +append --spreadsheet-id ID --range "Sheet1" --values "a,b,c"

# Slides
gws slides presentations create --title "New Deck"
```
```

#### `groups/main/CLAUDE.md` — Add the same section (or a reference to global)

Main group loads its own CLAUDE.md directly, not global. Either duplicate the section or add:

```markdown
## Google Workspace Access

See `/workspace/project/groups/global/CLAUDE.md` for Google Workspace CLI reference and permissions.
```

### Step 7: Update existing group settings.json files

Manually add the deny list to each existing group's settings:

- `data/sessions/slack_main/.claude/settings.json`
- `data/sessions/telegram_main/.claude/settings.json`
- `data/sessions/telegram_personal/.claude/settings.json`

Or delete them and let `container-runner.ts` regenerate with the new defaults on next spawn.

### Step 8: Rebuild and restart

```bash
./container/build.sh                          # Rebuild container image with gws
npm run build                                  # Compile TypeScript changes
systemctl --user restart nanoclaw              # Restart service
```

### Step 9: Test

Send a message to any chat:

- "Check my recent emails" → agent should run `gws gmail +triage`
- "What's on my calendar today?" → agent should run `gws calendar +agenda`
- "Create a new Google Doc called Test" → agent should run `gws docs documents create`
- "Try to send an email" → should be blocked by deny list

Check logs: `tail -f logs/nanoclaw.log`

## Files Changed Summary

| File | Change |
|------|--------|
| `container/Dockerfile` | Add `npm install -g @googleworkspace/cli` |
| `src/container-runner.ts` | Mount `~/.config/gws/` into containers, update default settings.json with deny list |
| `container/agent-runner/src/index.ts` OR `container/Dockerfile` | Set `GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file` env var |
| `groups/global/CLAUDE.md` | Add Google Workspace access section |
| `groups/main/CLAUDE.md` | Add reference to global CLAUDE.md or duplicate section |
| `container/skills/gws-*` | 9 skill folders copied from gws-cli repo |
| `data/sessions/*/settings.json` | Add deny list to existing groups (manual) |

## Security Layers

1. **OAuth scopes** (Google-level): gmail.readonly, calendar.readonly, drive.file — Google API rejects unauthorized operations
2. **Deny list** (Claude Code-level): Blocks bash commands matching destructive patterns before execution
3. **CLAUDE.md instructions** (prompt-level): Tells the agent what it can and cannot do
4. **Read-only mount** (Docker-level): `~/.config/gws/` mounted read-only — agents can't modify auth credentials

## Credential Details

| File | Purpose | Required in container |
|------|---------|----------------------|
| `~/.config/gws/credentials.enc` | Encrypted OAuth credentials | Yes |
| `~/.config/gws/.encryption_key` | AES-256-GCM key to decrypt credentials | Yes |
| `~/.config/gws/client_secret.json` | GCP OAuth client config | Yes |
| `~/.config/gws/token_cache.json` | Cached access/refresh tokens | Yes |
| `~/.config/gws/cache/` | Discovery API cache | Optional (improves performance) |

The entire `~/.config/gws/` directory is mounted as a single read-only volume. The env var `GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file` tells `gws` to use the `.encryption_key` file instead of the OS keyring (unavailable in Docker).
