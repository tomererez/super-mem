---
name: super-mem
description: "Persistent cross-session context loading and daily work logging for any project. Gives any project a 'brain' - a structured knowledge base + daily work log that loads into every Claude session, solving the 'Claude starts cold' problem. Use this skill when: (1) user types /super-mem or asks about project memory setup, (2) user says 'session start' and wants to load project context, (3) user says 'session end' and wants to log what was done, (4) user says 'end of day' and wants a daily summary. Also use when discussing cross-session context, session logging, or persistent project knowledge."
---

# super-mem

> Persistent cross-session context for any project.

A SessionStart hook writes your project's full knowledge base + Claude memories + daily logs to disk on every session start. You say "session start" and Claude loads it all. You say "session end" and Claude logs what happened. Next session, the new Claude picks up right where you left off.

## Routing

Parse the user's input and route to the matching section below:

| Input | Go to |
|-------|-------|
| `/super-mem` (no args) | **Status & Help** |
| `/super-mem setup` | **Setup Flow** |
| `/super-mem expand <module>` | **Expand Module** |
| `session start` | **Trigger: Session Start** |
| `session end` | **Trigger: Session End** |
| `end of day` | **Trigger: End of Day** |

---

## Status & Help

Check if `docs/brain/` (or custom path) exists in the project root. Check if a hook is registered in `~/.claude/settings.json`. Show:

```
super-mem - Persistent cross-session context

Commands:
  /super-mem setup          Set up brain for this project
  /super-mem expand <mod>   Add a module (team, domain, operations, concepts, reference)

Triggers:
  "session start"           Load full project context
  "session end"             Log session summary to daily file
  "end of day"              Session summary + team daily summary

Status:
  Brain: <path or "not set up">
  Hook: <registered or "not registered">
```

---

## Setup Flow

Interactive setup. Ask the user these questions using AskUserQuestion:

### 1. Project name
Auto-detect from `package.json` name field, `Cargo.toml`, `go.mod`, or directory name. Confirm with user.

### 2. Project root
Auto-detect from cwd. Confirm with user.

### 3. Working alone or with a team?
Options: Solo / Team (2+)

### 4. Complex business logic?
(financial calculations, legal rules, multi-step workflows)
Options: No / Yes

### 5. Deploy to production?
Options: Not yet / Yes

### Create the brain

After collecting answers, create the directory structure. Read template content from the `references/templates/` directory within this skill's directory for each file.

**Always created (base):**
```
docs/brain/
  status.md              <- from references/templates/base/status.md
  decisions.md           <- from references/templates/base/decisions.md
  current-focus.md       <- from references/templates/base/current-focus.md
  daily/                 <- empty directory, auto-populated by hook
```

**If Team:** also create `docs/brain/team/` and `docs/brain/workflows/` from their templates.

**If Complex business logic:** also create `docs/brain/domain/` and `docs/brain/concepts/` from their templates.

**If Production:** also create `docs/brain/operations/` and `docs/brain/reference/` from their templates.

### Generate the hook

1. Read `scripts/context-loader-template.js` from this skill's directory
2. Replace these placeholders with actual values:
   - `__PROJECT_ROOT__` - absolute path to project root (use forward slashes on all platforms)
   - `__PROJECT_NAME__` - project name from step 1
   - `__BRAIN_DIR__` - relative path from project root to brain (default: `docs/brain`)
   - `__MEMORY_DIR__` - Claude's auto-memory directory for this project. Find it at `~/.claude/projects/` - look for a directory whose name contains the project root path (lowercased, separators replaced with dashes). The memory dir is `<that-dir>/memory/`.
3. Write the customized hook to `~/.claude/hooks/<project-name>-context-loader.js`

### Register the hook

Read `~/.claude/settings.json`. Add the hook to `hooks.SessionStart`:

```json
{
  "type": "command",
  "command": "node \"<absolute-path-to-hook>\""
}
```

If `hooks.SessionStart` already has entries, append to the existing array. Do NOT overwrite other hooks.

### Finalize

1. Add `.claude/loaded-context.md` to the project's `.gitignore`
2. Save a feedback memory so future sessions know the trigger phrases:
   ```
   "session start" -> Read .claude/loaded-context.md
   "session end" -> Append summary to docs/brain/daily/YYYY-MM-DD.md
   "end of day" -> Summary + team daily summary
   ```
3. Show summary to user with brain path, hook path, and trigger phrases

---

## Expand Module

`/super-mem expand <module>` adds a module after initial setup.

| Module | Creates | Templates from |
|--------|---------|---------------|
| `team` | `docs/brain/team/` | `references/templates/team/` |
| `workflows` | `docs/brain/workflows/` | `references/templates/team/workflows.md` |
| `domain` | `docs/brain/domain/` | `references/templates/domain/` |
| `concepts` | `docs/brain/concepts/` | `references/templates/domain/concepts.md` |
| `operations` | `docs/brain/operations/` | `references/templates/operations/` |
| `reference` | `docs/brain/reference/` | `references/templates/reference/` |

Read template content from the skill's `references/templates/` directory. Create the directory and files. Confirm to user what was created.

---

## Trigger: Session Start

When the user says "session start":

1. Read `.claude/loaded-context.md` from the project root
2. Briefly confirm what was loaded:
   ```
   Context loaded: Brain (N files) | Memory (N files) | Daily (N days) | ~NK tokens
   ```
3. If the file doesn't exist, tell the user to start a new session so the hook can build it, then say "session start" again.

---

## Trigger: Session End

When the user says "session end":

1. Determine today's date (YYYY-MM-DD) and current time (HH:MM)
2. Find or create today's daily file at `docs/brain/daily/YYYY-MM-DD.md`
3. Check what session number this is by counting `#### Session N` entries in the file
4. Check git status and recent commits for the summary
5. Append a session summary:

```markdown
#### Session N | HH:MM-HH:MM
**Topics:** topic1, topic2
- what was done (technical level)
- decisions made and why
**Files:** changed files
**Commits:** abc1234 or (none)
**Open:** unfinished items
**Decisions:** key decisions with reasoning
```

### Repeat "session end" in same session

If the user says "session end" again and an entry for the current session already exists, DON'T create a new entry. Append a continuation:

```markdown
**Continued | HH:MM-HH:MM**
- additional work done
**Files:** additional changed files
```

---

## Trigger: End of Day

Does everything "session end" does, then generates a team-facing daily summary. Default format:

```markdown
## Daily Summary - YYYY-MM-DD

### What was done
- bullet points summarizing all sessions

### Key decisions
- decisions and reasoning

### Tomorrow
- planned next steps
```

The user can customize this format. If they have a preferred summary format (bilingual, specific structure), follow their instructions.

---

## How it works

```
Session start (hook runs automatically)
  -> Hook collects: Brain .md files + Memory .md files + Daily logs (14 days)
  -> Hook writes combined content to .claude/loaded-context.md (no size limit)
  -> Hook appends "Session N | HH:MM" to today's daily file
  -> Hook returns small status via additionalContext (~200 chars)

User says "session start"
  -> Claude reads .claude/loaded-context.md (full context, ~50K tokens typical)
  -> Claude confirms what was loaded

User works...

User says "session end"
  -> Claude appends summary to docs/brain/daily/YYYY-MM-DD.md
```

This bypasses the 10KB `additionalContext` limit by writing full content to disk and reading on demand.

## Token budget

| Component | Typical size | % of 1M |
|-----------|-------------|---------|
| Brain (small, ~10 files) | ~5K tokens | 0.5% |
| Brain (medium, ~30 files) | ~15K tokens | 1.5% |
| Brain (large, ~65 files) | ~30K tokens | 3% |
| Memory (typical) | ~20K tokens | 2% |
| Daily log (14 days) | ~10K tokens | 1% |
| **Total (medium)** | **~45K tokens** | **4.5%** |

Target: under 10% of context window.
