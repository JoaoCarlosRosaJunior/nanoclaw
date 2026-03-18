# 003 — Agent Template System

## Goal

Each NanoClaw agent has a self-contained template defining its personality, skills, Docker image, environment, and permissions. When a group is registered, it references a template by name. At container spawn time, NanoClaw reads the template and configures the container accordingly. No manual editing of settings.json, no manual skill copying, no shared-everything default.

This is the foundation for running multiple specialized agents — engineering, financial, teacher, coach — each with different tools, credentials, and instructions, all from a single NanoClaw installation.

## Architecture Overview

```
agents/                              ← agent templates (in git)
├── _base/
│   ├── agent.yaml                   ← manifest: image, skills, settings
│   └── CLAUDE.md                    ← global personality
├── engineering/
│   ├── agent.yaml
│   └── CLAUDE.md
└── financial/
    ├── agent.yaml
    └── CLAUDE.md

container/docker/                    ← all Dockerfiles (in git)
├── Dockerfile                       ← base image
├── Dockerfile.engineering           ← FROM base + boto3, pymongo
└── Dockerfile.financial             ← FROM base + libreoffice

container/skills/                    ← ALL skills, single source of truth
├── xlsx/
├── pdf/
├── gws-gmail/
├── mongo-queries/                   ← agent-specific skill, still lives here
└── ...

~/.config/nanoclaw/agents/           ← secrets (outside git, per-machine)
├── engineering/
│   └── secrets.env                  ← MONGO_URI=..., AWS_ACCESS_KEY_ID=...
└── financial/
    └── secrets.env

registered_groups (SQLite):          ← runtime binding: where → what
JID               | folder              | agent
tg:-5294239484    | telegram_main       | _base
slack:C0ENGINEER  | slack_engineering   | engineering
tg:-100xxx        | telegram_financial  | financial
```

## agent.yaml Manifest Format

```yaml
# agents/engineering/agent.yaml
name: Engineering Agent

# Docker image — built from the referenced Dockerfile
image: nanoclaw-engineering:latest
dockerfile: container/docker/Dockerfile.engineering

# Skills to load — references to container/skills/{name}
# ONLY these skills are copied into the container. No base inheritance, no exclude.
skills:
  - agent-browser
  - gws-shared
  - gws-gmail
  - gws-drive
  - gws-docs
  - pdf
  - xlsx
  - mongo-queries

# Settings overrides — merged with runtime settings at spawn time
settings:
  model: claude-sonnet-4-6
  permissions:
    deny:
      - "Bash(gws gmail * send *)"
      - "Bash(gws gmail * delete *)"
      - "Bash(rm -rf /*)"

# Environment variables — schema + secrets file path
env:
  required:
    - MONGO_URI
    - AWS_ACCESS_KEY_ID
    - AWS_SECRET_ACCESS_KEY
  file: ~/.config/nanoclaw/agents/engineering/secrets.env

# Additional mounts — host directories exposed to the container
mounts:
  - hostPath: ~/.aws
    containerPath: /home/node/.aws
    readonly: true
```

```yaml
# agents/_base/agent.yaml
name: Base Agent

image: nanoclaw-agent:latest
dockerfile: container/docker/Dockerfile

skills:
  - agent-browser
  - gws-shared
  - gws-gmail
  - gws-gmail-triage
  - gws-drive
  - gws-docs
  - gws-sheets
  - gws-slides
  - gws-calendar
  - gws-workflow
  - pdf
  - xlsx
  - docx
  - pptx

settings:
  permissions:
    deny:
      - "Bash(gws gmail * send *)"
      - "Bash(gws gmail * delete *)"
      - "Bash(gws gmail * trash *)"
      - "Bash(gws gmail * modify *)"
      - "Bash(gws gmail * insert *)"
      - "Bash(gws drive * delete *)"
      - "Bash(gws drive * trash *)"
      - "Bash(gws calendar * delete *)"
      - "Bash(gws calendar * insert *)"
      - "Bash(gws calendar * update *)"
      - "Bash(gws calendar * patch *)"
      - "Bash(gws docs * delete *)"
      - "Bash(gws sheets * delete *)"
      - "Bash(gws slides * delete *)"
```

```yaml
# agents/financial/agent.yaml
name: Financial Agent

image: nanoclaw-financial:latest
dockerfile: container/docker/Dockerfile.financial

skills:
  - agent-browser
  - gws-shared
  - gws-sheets
  - gws-drive
  - gws-docs
  - xlsx
  - pdf

settings:
  model: claude-opus-4-6
  permissions:
    deny:
      - "Bash(gws gmail * send *)"
      - "Bash(gws gmail * delete *)"
      - "Bash(rm -rf /*)"

env:
  required: []
  file: null
```

## CLAUDE.md Ownership

Two separate files serve different purposes:

| File | Purpose | Lifecycle |
|------|---------|-----------|
| `agents/{name}/CLAUDE.md` | **Template personality** — role, instructions, what the agent can/cannot do | Written once by `/create-agent`. Updated manually when the role changes. Copied into the container as system prompt `append` on every spawn. |
| `groups/{folder}/CLAUDE.md` | **Runtime memory** — agent-written notes, learned preferences, accumulated knowledge | Created by the agent. Persists across sessions. Never overwritten by the template. Loaded as project-level CLAUDE.md since container `cwd` is `/workspace/group`. |

Both are loaded by the SDK — the template as system prompt append, the runtime as project-level CLAUDE.md. They complement each other.

## Channel/Group Strategy

Agent templates define **what** the agent is. Group registrations define **where** it listens.

| Where | Agent | How |
|-------|-------|-----|
| Telegram DM | _base (main) | Existing registration |
| Telegram group (MyAgents) | _base (main + swarm) | Existing registration |
| Slack DM | _base (main) | Existing registration |
| Slack #engineering channel | engineering | Create channel → add bot → register with `agent: engineering` |
| Telegram "Finance" group | financial | Create group → add bot → register with `agent: financial` |

All channels use the **same** Telegram bot and Slack bot. The bot is just transport — it doesn't know or care about agents. The `agent` field on the registered group determines which template is loaded.

**DMs are limited to one agent** — you can't have two DMs with the same bot. The DM is the main/general agent. For specialized agents, use groups or channels.

**Future consideration**: Calling agents by `@name` in a shared group (routing layer). Not needed now. Would require a routing layer on top of the current message detection that maps `@Engineering` to the engineering agent template.

## Files to Read Before Implementing

### Core files that need modification

1. **`src/container-runner.ts`** — The main file to modify. `buildVolumeMounts()` (line 59-176) builds the mount list. `runContainerAgent()` (line 267+) spawns the container with a hardcoded `CONTAINER_IMAGE`. Both need to read from the agent template instead.

2. **`src/db.ts`** — `registered_groups` table schema (line 26-34). Add `agent TEXT DEFAULT '_base'` column. Update `setRegisteredGroup()` and `getAllRegisteredGroups()`.

3. **`src/types.ts`** — `RegisteredGroup` interface (line 35-43). Add `agent?: string` field.

4. **`container/build.sh`** — Currently builds one Dockerfile. Needs to scan `agents/*/agent.yaml`, validate, and build all referenced Dockerfiles.

### Files to understand (read but don't modify much)

5. **`src/config.ts`** — Paths (`GROUPS_DIR`, `DATA_DIR`). May need an `AGENTS_DIR` constant.

6. **`src/index.ts`** — `main()` function (line 473+) creates channels and starts subsystems. The agent template resolution happens before container spawn, so this file may not need changes beyond passing the template info through.

7. **`container/agent-runner/src/index.ts`** — How skills are loaded (via `.claude/skills/`), how env vars reach the SDK. Understanding this helps design how template env vars flow into the container.

8. **`container/Dockerfile`** — Current base Dockerfile. Will move to `container/docker/Dockerfile`. Agent-specific Dockerfiles inherit from it.

## Implementation Steps

### Step 1: Create new `src/agent-resolver.ts`

New file — no core file modification. This module:

- Reads and parses `agent.yaml` from `agents/{name}/`
- Validates all listed skills exist in `container/skills/`
- Validates secrets file exists (if `env.file` is set)
- Validates all required env vars are present in the secrets file
- Reads and parses settings overrides
- Returns a resolved `AgentTemplate` object

```typescript
export interface AgentTemplate {
  name: string;
  image: string;
  dockerfile?: string;
  skills: string[];
  settings: {
    model?: string;
    permissions?: { deny?: string[] };
    env?: Record<string, string>;
  };
  envVars: Record<string, string>;  // resolved from secrets file
  mounts: Array<{ hostPath: string; containerPath: string; readonly: boolean }>;
  claudeMdPath: string;  // path to the template's CLAUDE.md
}

export function resolveAgentTemplate(agentName: string): AgentTemplate;
```

**Validation errors should throw with descriptive messages:**
- `AgentTemplateError: Skill "mongo-queries" not found in container/skills/. Agent: engineering`
- `AgentTemplateError: Secrets file not found: ~/.config/nanoclaw/agents/engineering/secrets.env. Create it with: MONGO_URI=..., AWS_ACCESS_KEY_ID=..., AWS_SECRET_ACCESS_KEY=...`
- `AgentTemplateError: Missing required env var "MONGO_URI" in ~/.config/nanoclaw/agents/engineering/secrets.env`
- `AgentTemplateError: Agent template "engineering" not found in agents/. Available: _base, financial`

If the agent name is `_base` or undefined, load from `agents/_base/`.

### Step 2: Modify `container-runner.ts` to use agent templates

Read the agent template at the start of `runContainerAgent()`:

```typescript
import { resolveAgentTemplate } from './agent-resolver.js';

// In runContainerAgent():
const agentName = group.agent || '_base';
const template = resolveAgentTemplate(agentName);

// Use template.image instead of CONTAINER_IMAGE
// Use template.skills to copy only listed skills
// Use template.envVars for Docker -e flags
// Use template.mounts for additional -v flags
// Use template.settings to generate settings.json
// Use template.claudeMdPath for the system prompt
```

**Settings merge logic** (template defaults + runtime overrides):

```typescript
// 1. Start with template settings
const settings = { ...template.settings };

// 2. Overlay runtime settings (data/sessions/{folder}/.claude/settings.json)
const runtimeSettingsPath = path.join(groupSessionsDir, 'settings.json');
if (fs.existsSync(runtimeSettingsPath)) {
  const runtime = JSON.parse(fs.readFileSync(runtimeSettingsPath, 'utf-8'));
  // Runtime model overrides template model
  if (runtime.model) settings.model = runtime.model;
  // Runtime env merges with template env
  if (runtime.env) settings.env = { ...settings.env, ...runtime.env };
  // Runtime deny list extends template deny list (not replaces)
  // ... decide merge strategy
}

// 3. Write merged settings to the group's .claude/settings.json
```

**Key decision on settings merge**: Should runtime deny list **replace** or **extend** the template deny list? Recommendation: **extend** (union). The template sets the minimum security baseline; runtime can add more restrictions but not remove template restrictions. This prevents an agent from weakening its own security via the model selection CLAUDE.md trick.

**Skills copying changes:**

Currently (line 149-159), skills are copied from `container/skills/` to all groups. Change to:

```typescript
// Only copy skills listed in the template
for (const skillName of template.skills) {
  const srcDir = path.join(process.cwd(), 'container', 'skills', skillName);
  // Validation already done in resolveAgentTemplate, but double-check
  if (!fs.existsSync(srcDir)) {
    throw new Error(`Skill "${skillName}" not found`);
  }
  const dstDir = path.join(skillsDst, skillName);
  fs.cpSync(srcDir, dstDir, { recursive: true });
}
```

**System prompt changes:**

Currently, non-main groups get `groups/global/CLAUDE.md` as system prompt append. Change to load from the agent template's CLAUDE.md instead:

```typescript
// In the container input or agent-runner:
// System prompt = agents/{agentName}/CLAUDE.md (template personality)
// Project-level CLAUDE.md = groups/{folder}/CLAUDE.md (runtime memory)
```

### Step 3: Add `agent` field to registered_groups

**File:** `src/db.ts`

Add migration:
```sql
ALTER TABLE registered_groups ADD COLUMN agent TEXT DEFAULT '_base';
```

**File:** `src/types.ts`

```typescript
export interface RegisteredGroup {
  // ... existing fields ...
  agent?: string;  // agent template name, defaults to '_base'
}
```

Update `setRegisteredGroup()`, `getAllRegisteredGroups()`, and the setup `register` step to include the `agent` field.

### Step 4: Move Dockerfile and update build.sh

Move `container/Dockerfile` → `container/docker/Dockerfile`.

Update `build.sh` to:
1. Scan all `agents/*/agent.yaml` files
2. Extract `image` and `dockerfile` fields
3. **Validate**: same Dockerfile → must have same image name. Same image name → must have same Dockerfile. Fail with descriptive error on conflict.
4. Deduplicate (don't build the same Dockerfile twice)
5. Build base first (the one without `FROM nanoclaw-*`)
6. Build dependent images in dependency order
7. Report results

```bash
# Example build.sh output:
# Building base: container/docker/Dockerfile → nanoclaw-agent:latest ... OK
# Building engineering: container/docker/Dockerfile.engineering → nanoclaw-engineering:latest ... OK
# Building financial: container/docker/Dockerfile.financial → nanoclaw-financial:latest ... OK
# All 3 images built successfully.
```

Support building a single agent's image:
```bash
./container/build.sh engineering   # build only the engineering image (+ base if needed)
./container/build.sh               # build all
```

### Step 5: Environment variable injection

At spawn time, `container-runner.ts` reads the secrets file and passes env vars to the container via Docker `-e` flags:

```typescript
const envFlags: string[] = [];
for (const [key, value] of Object.entries(template.envVars)) {
  envFlags.push('-e', `${key}=${value}`);
}
// Add to the docker run command
```

**Security considerations:**
- Secrets file is read-only, outside git (`~/.config/nanoclaw/agents/`)
- Docker `-e` flags are visible in `docker inspect` and `/proc/*/environ` — this is the same security level as the existing credential proxy pattern
- The deny list can prevent the agent from reading its own env vars via Bash:
  ```json
  "deny": ["Bash(*secrets*)", "Bash(*environ*)"]
  ```
- For higher security (future): extend the credential proxy pattern to handle non-Claude credentials

### Step 6: Create `/create-agent` skill

**File:** `.claude/skills/create-agent/SKILL.md`

Interactive wizard that:

1. **Ask purpose**: "What should this agent do?" → Generate initial CLAUDE.md with personality, role, instructions, and capabilities section

2. **List available skills**: Show all skills in `container/skills/` with descriptions. Ask user to pick which ones this agent needs.

3. **Ask about credentials**: "Does this agent need any credentials or API keys?" → If yes, collect the env var names (not values). Generate the `env` section in agent.yaml.

4. **Ask about file access**: "Does this agent need access to any host directories?" → Configure mounts.

5. **Ask about Docker image**: "Does this agent need any tools beyond the base image?" → If yes, create a `Dockerfile.{name}` in `container/docker/` that inherits from the base. Ask what to install (`pip install`, `apt install`, `npm install`).

6. **Ask about model**: "Which Claude model should this agent use by default?" → Set in settings.

7. **Ask about permissions**: "Are there any operations this agent should be blocked from?" → Generate deny list. Pre-populate with sensible defaults from the base template.

8. **Generate files**:
   - `agents/{name}/agent.yaml`
   - `agents/{name}/CLAUDE.md`
   - `container/docker/Dockerfile.{name}` (if custom)
   - `~/.config/nanoclaw/agents/{name}/secrets.env` (empty, with comments listing required vars)

9. **Build image**: Run `./container/build.sh {name}`

10. **Ask about channels**: "Where should this agent listen?"
    - Create a Telegram group? → guide user through creation, add bot, `/chatid`
    - Create a Slack channel? → guide user through creation, add bot
    - Both?

11. **Register group(s)**: Use `npx tsx setup/index.ts --step register` with `--agent {name}`

12. **Restart service**: `systemctl --user restart nanoclaw`

13. **Test**: Tell user to send a message in the new channel/group.

### Step 7: Update `/update-nanoclaw` skill

Add a note warning about potential conflicts in:
- `src/container-runner.ts` (agent template resolution)
- `src/db.ts` (agent column)
- `src/types.ts` (agent field)
- `container/build.sh` (multi-image build)

Guide the user through resolving conflicts in these files if they occur after `git merge upstream/main`.

## Tests

### Unit tests for `agent-resolver.ts`

```
✓ resolves _base template with correct defaults
✓ resolves named template with all fields
✓ throws on missing skill in include list
✓ throws on missing agent template folder
✓ throws on missing secrets file when env.file is set
✓ throws on missing required env var in secrets file
✓ returns empty envVars when no env section
✓ returns empty mounts when no mounts section
✓ falls back to _base when agent name is undefined
✓ reads CLAUDE.md path correctly
✓ handles agent.yaml without optional fields (dockerfile, env, mounts)
```

### Unit tests for settings merge

```
✓ template settings used when no runtime overrides exist
✓ runtime model overrides template model
✓ runtime deny list extends (not replaces) template deny list
✓ runtime env merges with template env
✓ template deny list cannot be removed by runtime settings
```

### Unit tests for skills resolution

```
✓ only listed skills are copied to container
✓ skills not in the list are NOT copied
✓ agent-specific skills in container/skills/ are resolved correctly
✓ empty skills list results in no skills copied
```

### Integration tests for container-runner

```
✓ container spawns with correct image from agent template
✓ container has only the listed skills mounted
✓ container has env vars from secrets file
✓ container has additional mounts from template
✓ container settings.json reflects merged template + runtime
```

### Build script tests

```
✓ builds base image first
✓ builds dependent images after base
✓ errors on same Dockerfile with different image names
✓ errors on same image name with different Dockerfiles
✓ skips agents without dockerfile field (they use base)
✓ supports building single agent by name
✓ reports build results for all images
```

### Registration tests

```
✓ group registered with agent field stores correctly in SQLite
✓ group without agent field defaults to _base
✓ group with non-existent agent name → error at spawn, not at registration
```

## Migration Path

### Existing installation

1. Create `agents/_base/` folder with `agent.yaml` and `CLAUDE.md` (extracted from current `groups/global/CLAUDE.md`)
2. Move `container/Dockerfile` → `container/docker/Dockerfile`
3. Update `build.sh`
4. Add `agent` column to `registered_groups` (defaults to `_base`)
5. Existing groups continue to work — `_base` template matches current behavior

### Backward compatibility

- Groups without an `agent` field use `_base` — no breakage
- `container/skills/` stays as the single skill location — no change for existing skills
- Base Dockerfile moves but is still built the same way
- Settings merge is additive — existing runtime settings.json files are preserved

## Files Changed Summary

| File | Change | New or Modified |
|------|--------|-----------------|
| `src/agent-resolver.ts` | Agent template resolution logic | **New** |
| `src/agent-resolver.test.ts` | Tests for template resolution | **New** |
| `src/container-runner.ts` | Read agent template at spawn, use template image/skills/env/settings | Modified |
| `src/db.ts` | Add `agent` column to `registered_groups` | Modified |
| `src/types.ts` | Add `agent?: string` to `RegisteredGroup` | Modified |
| `src/config.ts` | Add `AGENTS_DIR` constant | Modified |
| `container/build.sh` | Multi-Dockerfile scanning, validation, build | Modified (significant) |
| `container/docker/Dockerfile` | Moved from `container/Dockerfile` | Moved |
| `agents/_base/agent.yaml` | Base agent manifest | **New** |
| `agents/_base/CLAUDE.md` | Base personality (from current global) | **New** |
| `.claude/skills/create-agent/SKILL.md` | Interactive agent creation wizard | **New** |

## Design Constraints

1. **Prefer new files over modifying existing core files** — `agent-resolver.ts` is new. Minimizes upstream merge conflicts.
2. **Skills referenced by name, never duplicated** — single source in `container/skills/`.
3. **No `exclude` for skills** — only explicit `include`. What you list is what you get.
4. **Fail loudly at spawn time** — missing skill, missing secrets, missing image → error with descriptive message. Never silently fall back.
5. **Template CLAUDE.md = personality (re-applied on spawn). Group CLAUDE.md = runtime memory (persists).**
6. **Secrets outside git** — `~/.config/nanoclaw/agents/{name}/secrets.env`.
7. **Template deny list is a baseline** — runtime can extend but not weaken.
8. **JIDs are NOT in agent.yaml** — templates are portable. JID → agent binding is in `registered_groups`.
9. **Same Dockerfile must have same image name** — build.sh validates, fails on conflict.
10. **One Docker image per agent is optional** — agents without `dockerfile` field use the base image.
