---
name: add-gws
description: Add Google Workspace CLI integration. Agents can read Gmail, view Calendar, and create/edit Drive, Docs, Sheets, and Slides via the gws CLI. Guides through installation, authentication, and security configuration.
---

# Add Google Workspace CLI Integration

This skill adds Google Workspace access to NanoClaw agents via the `gws` CLI tool. Agents use it through Bash — no MCP server needed. Security is enforced at three layers: OAuth scopes, Claude Code deny list, and CLAUDE.md instructions.

## Phase 1: Pre-flight

### Check if already applied

```bash
grep -q "googleworkspace/cli" container/Dockerfile && echo "Already applied" || echo "Not applied"
```

If already applied, skip to Phase 4 (Configure Authentication).

### Check prerequisites

AskUserQuestion: Do you have the following ready?

- **Node.js 18+** — required for npm install
- **A Google Cloud project** — required for OAuth credentials (can be created during `gws auth setup`)
- **A Google account** with access to Google Workspace

If they don't have a Google Cloud project, `gws auth setup` will walk them through creating one.

## Phase 2: Install gws on the host

Tell the user:

> First, let's install the Google Workspace CLI on your machine:
>
> ```bash
> npm install -g @googleworkspace/cli
> ```
>
> Verify it's installed:
> ```bash
> gws --version
> ```

Wait for confirmation.

### Alternative installation methods

If npm doesn't work, offer alternatives:

```bash
# Homebrew (macOS/Linux)
brew install googleworkspace-cli

# Pre-built binaries
# Download from: https://github.com/googleworkspace/cli/releases
```

## Phase 3: Apply Code Changes

### Merge the skill branch

```bash
git fetch origin skill/gws
git merge origin/skill/gws || {
  git checkout --theirs package-lock.json
  git add package-lock.json
  git merge --continue
}
```

This merges in:
- `container/Dockerfile` — installs `@googleworkspace/cli` and sets `GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file`
- `src/container-runner.ts` — mounts `~/.config/gws/` read-only into containers, adds default deny list for destructive operations
- `container/skills/gws-*` — 9 agent skill folders (gmail, drive, docs, sheets, slides, calendar, gmail-triage, shared, workflow)
- `groups/global/CLAUDE.md` — Google Workspace permissions and quick reference
- `groups/main/CLAUDE.md` — reference to global gws docs

If the merge reports conflicts, resolve them by reading the conflicted files and understanding the intent of both sides.

### Validate

```bash
npm run build
npm test
```

### Rebuild container

```bash
./container/build.sh
```

This installs `gws` in the container image. Takes a few minutes.

## Phase 4: Configure Authentication

### Run gws auth setup

Tell the user:

> Now let's authenticate with Google. This will open a browser for OAuth:
>
> ```bash
> gws auth setup
> ```
>
> Follow the prompts:
> 1. It will create or select a Google Cloud project
> 2. Enable the required APIs
> 3. Open a browser for OAuth consent
> 4. If you see "app isn't verified", click **Advanced** → **Go to [app name] (unsafe)** — this is normal for personal OAuth apps

### Scope selection

AskUserQuestion: What Google Workspace access do you need?

- **Read-only (recommended for safety)** — Read emails, view calendar, read files from Drive/Docs/Sheets/Slides. Cannot create, modify, or delete anything.
- **Read + Create** — Everything above, plus create and edit Docs, Sheets, Slides, and upload to Drive. Cannot delete or send emails.
- **Full access** — Everything. Use with caution.

Based on the user's choice, guide them to authenticate with the appropriate scopes:

**Read-only:**
```bash
gws auth login --scopes gmail.readonly,calendar.readonly,drive.readonly
```

**Read + Create (recommended):**
```bash
gws auth login --scopes gmail.readonly,calendar.readonly,calendar.events.readonly,calendar.freebusy,calendar.settings.readonly,drive.readonly,drive.file,drive.appdata,documents,spreadsheets,presentations
```

**Full access:**
```bash
gws auth login
```

### Verify authentication

```bash
gws gmail +triage
```

This should show the user's recent unread emails. If it fails, check:
- OAuth consent was completed in the browser
- The Google account has Gmail access
- Run `gws auth login` again if needed

### Verify credentials exist

```bash
ls ~/.config/gws/credentials.enc && ls ~/.config/gws/.encryption_key && echo "Credentials OK" || echo "Missing credentials"
```

Both files must exist. The container needs them to authenticate.

## Phase 5: Security Configuration

### Deny list

The default `settings.json` template (in `src/container-runner.ts`) already includes a deny list blocking destructive operations:

- Gmail: send, delete, trash, modify, insert
- Drive: delete, trash
- Calendar: delete, insert, update, patch
- Docs/Sheets/Slides: delete

### Update existing groups

Existing groups already have `settings.json` files that were created before the deny list was added. Delete them so they regenerate with the new defaults:

```bash
# List existing group settings
find data/sessions -name "settings.json" -path "*/.claude/*"

# Delete them (they'll regenerate on next container spawn)
find data/sessions -name "settings.json" -path "*/.claude/*" -delete
```

### Custom deny rules

AskUserQuestion: Do you want to customize the deny list? The defaults block delete/send/modify operations. You can add more restrictions or remove some.

If the user wants changes, edit `src/container-runner.ts` and find the `permissions.deny` array in the default settings template.

## Phase 6: Additional Skills

The default installation includes 9 gws skills:

| Skill | What it covers |
|-------|---------------|
| `gws-shared` | Auth, flags, CLI syntax (required by all others) |
| `gws-gmail` | Full Gmail API reference |
| `gws-gmail-triage` | Inbox summary helper |
| `gws-drive` | Drive files, folders, permissions |
| `gws-docs` | Google Docs read/write |
| `gws-sheets` | Google Sheets read/write/append |
| `gws-slides` | Google Slides create/edit |
| `gws-calendar` | Calendar events, agenda, freebusy |
| `gws-workflow` | Multi-service workflows (weekly digest, meeting prep) |

AskUserQuestion: Do you want to add more gws skills? The Google Workspace CLI repo has 80+ skills including recipes and personas.

If yes, tell the user:

> Clone the gws CLI repo to browse available skills:
> ```bash
> git clone https://github.com/googleworkspace/cli.git /tmp/gws-cli
> ls /tmp/gws-cli/skills/
> ```
>
> Copy any additional skills you want:
> ```bash
> cp -r /tmp/gws-cli/skills/SKILL_NAME container/skills/
> ```
>
> Skills are auto-loaded into all groups on next container spawn — no restart needed.

## Phase 7: CLAUDE.md Updates

### Check group CLAUDE.md files

The skill branch adds Google Workspace instructions to `groups/global/CLAUDE.md` and `groups/main/CLAUDE.md`. Check if other main groups need the same:

```bash
ls groups/*/CLAUDE.md
```

For each main group that doesn't reference the global CLAUDE.md, add:

```markdown
## Google Workspace Access

See `/workspace/project/groups/global/CLAUDE.md` for Google Workspace CLI reference and permissions.
```

### Acknowledgment messages

For better UX, add to each main group's CLAUDE.md:

```markdown
When you receive a request that will take more than a few seconds, ALWAYS send a brief acknowledgment first using `mcp__nanoclaw__send_message` before starting the work (e.g., "On it, checking your emails..." or "Looking into that now...").
```

## Phase 8: Restart and Test

```bash
systemctl --user restart nanoclaw   # Linux
# macOS: launchctl kickstart -k gui/$(id -u)/com.nanoclaw
```

### Test commands

Send these messages in any registered chat:

1. **Gmail:** "Check my recent emails" → agent should run `gws gmail +triage`
2. **Calendar:** "What's on my calendar today?" → agent should run `gws calendar +agenda`
3. **Docs:** "Create a new Google Doc called Test" → agent should run `gws docs documents create`
4. **Sheets:** "Create a spreadsheet with headers Name, Email, Phone" → agent should create a sheet
5. **Deny list test:** "Send an email to test@example.com" → should be blocked

Check logs: `tail -f logs/nanoclaw.log`

## Troubleshooting

### gws command not found inside container

1. Verify container was rebuilt: `./container/build.sh`
2. Check image has gws: `docker run --rm nanoclaw-agent:latest gws --version`

### Authentication fails inside container

1. Check credentials are mounted: look for `mountCount: 7` (or higher) in nanoclaw.log when container spawns
2. Verify encryption key exists: `ls ~/.config/gws/.encryption_key`
3. Check env var: `docker run --rm nanoclaw-agent:latest env | grep KEYRING` — should show `GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file`
4. If using OS keyring on host, switch to file: `GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file gws auth login`

### Token expired

Re-authenticate:
```bash
gws auth login
systemctl --user restart nanoclaw   # restart to pick up new token
```

### Deny list not working

1. Check `data/sessions/{group}/.claude/settings.json` has the `permissions.deny` array
2. If missing, delete the file and restart — it will regenerate from the template
3. Verify: `cat data/sessions/{group}/.claude/settings.json | grep deny`

### Agent not using gws

1. Check skills are present: `ls data/sessions/{group}/.claude/skills/ | grep gws`
2. Check CLAUDE.md mentions Google Workspace: `grep -i "google workspace\|gws" groups/global/CLAUDE.md`
3. Try explicit instructions: "Use the gws CLI to check my inbox"

## Removal

1. Remove gws from Dockerfile:
   - Delete the `npm install -g ... @googleworkspace/cli` line
   - Delete the `ENV GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file` line
2. Remove mount from `src/container-runner.ts` (the `gwsConfigDir` block)
3. Remove deny list from `src/container-runner.ts` (the `permissions.deny` array)
4. Remove skills: `rm -r container/skills/gws-*`
5. Remove Google Workspace section from `groups/global/CLAUDE.md` and other group CLAUDE.md files
6. Rebuild: `./container/build.sh && npm run build && systemctl --user restart nanoclaw`
