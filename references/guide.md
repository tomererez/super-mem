# super-mem Implementation Guide

> How super-mem works under the hood. Read this for troubleshooting or manual setup.

## The Problem

Claude Code starts each session with no memory of previous work. The `additionalContext` field in SessionStart hooks is limited to 10,000 characters. Anything larger gets saved to a temp file with only a 2KB preview. For a full project context (50-200KB), the hook approach alone doesn't work.

## The Solution

```
                    SessionStart Hook
              (context-loader-template.js)
 ------------------------------------------------
  1. Collects all .md files from:
     Brain + Memory + Daily logs (14 days)
  2. Writes to .claude/loaded-context.md (no limit)
  3. Appends session entry to daily log
  4. Returns small additionalContext (stats only)
 ------------------------------------------------
                       |
           User says "session start"
                       |
                       v
         Claude reads loaded-context.md
         (~50K tokens typical, <10% of 1M)
```

## Components

### 1. The Brain (`docs/brain/`)

A structured markdown knowledge base. Organized by topic:

```
docs/brain/
  status.md              # Current project state
  decisions.md           # Append-only decision log
  current-focus.md       # What we're working on now
  daily/                 # Auto-populated session logs
  team/                  # (optional) People and roles
  workflows/             # (optional) Git flow, PR rules
  domain/                # (optional) Business rules
  concepts/              # (optional) Entity definitions
  operations/            # (optional) Deploy, incidents
  reference/             # (optional) External systems
```

### 2. The Hook (`~/.claude/hooks/<project>-context-loader.js`)

Runs on every SessionStart. Key functions:

- `walkDir(dir, excludeDirs)` - Recursive .md file collector
- `loadBrain()` - Loads all brain files except `daily/`
- `loadMemory()` - Loads Claude's auto-memory files for this project
- `loadDailyLogs()` - Loads daily files from the last 14 days
- `appendSessionStart()` - Appends `#### Session N | HH:MM` to today's daily file
- `main()` - Orchestrates everything, writes combined file, returns status

The hook only activates when `cwd` is under the project root.

### 3. The Trigger System

| Trigger | Action |
|---------|--------|
| Hook (automatic) | Builds context file + records session start |
| "session start" | Claude reads the context file |
| "session end" | Claude appends summary to daily log |
| "end of day" | Summary + team-facing daily summary |

### 4. The Daily Log (`docs/brain/daily/YYYY-MM-DD.md`)

One file per day, accumulates session summaries:

```markdown
# Daily Log 2026-05-25

#### Session 1 | 09:00
... (auto-appended by hook)

#### Session 1 | 09:00-10:30
**Topics:** auth, API
- Implemented JWT refresh
- Fixed token expiry bug
**Files:** src/auth.ts, src/api/middleware.ts
**Commits:** abc1234
**Open:** need to test with expired tokens
**Decisions:** chose refresh tokens over session extension

#### Session 2 | 11:00
... (auto-appended by hook for session 2)
```

## Manual Setup

If the `/super-mem setup` command isn't available, follow these steps:

### Step 1: Create the brain

```bash
mkdir -p docs/brain/daily
```

Create at minimum: `status.md`, `decisions.md`, `current-focus.md`.

### Step 2: Copy and customize the hook

Copy `scripts/context-loader-template.js` and replace the placeholder constants:

```js
const PROJECT_ROOT = 'C:\\Users\\...\\YourProject';
const PROJECT_NAME = 'YourProject';
const BRAIN_DIR = path.join(PROJECT_ROOT, 'docs', 'brain');
const MEMORY_DIR = 'C:\\Users\\...\\.claude\\projects\\<project-slug>\\memory';
```

Save to `~/.claude/hooks/<project>-context-loader.js`.

### Step 3: Register the hook

Add to `~/.claude/settings.json`:

```json
{
  "hooks": {
    "SessionStart": [{
      "hooks": [{
        "type": "command",
        "command": "node \"C:/Users/.../.claude/hooks/<project>-context-loader.js\""
      }]
    }]
  }
}
```

### Step 4: Gitignore

Add `.claude/loaded-context.md` to `.gitignore`.

### Step 5: Save trigger memories

Create feedback memories so Claude knows the trigger phrases in future sessions.

## Troubleshooting

### Context file not created
- Check that the hook is registered in `~/.claude/settings.json`
- Check that `cwd` matches the project root (case-insensitive)
- Run the hook manually: `node ~/.claude/hooks/<project>-context-loader.js`

### Context file is empty
- Check that `docs/brain/` exists and has `.md` files
- Check the memory directory path is correct

### Hook runs but no additionalContext visible
- The additionalContext is limited to 10KB. The hook should return a small status message only.
- If another hook outputs the same `hookEventName`, only the last one wins. Make sure hook event names don't collide.

### Daily log not being written
- Check that `docs/brain/daily/` directory exists
- The hook creates it with `mkdirSync({ recursive: true })` but check permissions

## Token Budget Reference

| Component | Typical size | % of 1M |
|-----------|-------------|---------|
| Brain (small, ~10 files) | ~5K tokens | 0.5% |
| Brain (medium, ~30 files) | ~15K tokens | 1.5% |
| Brain (large, ~65 files) | ~30K tokens | 3% |
| Memory (typical) | ~20K tokens | 2% |
| Daily log (14 days) | ~10K tokens | 1% |
| **Total (medium)** | **~45K tokens** | **4.5%** |

Rule of thumb: stay under 10% of context window for auto-loaded content.

## Limitations

- The 10KB limit on `additionalContext` is a Claude Code constraint (as of May 2026). This system works around it by writing to disk.
- No AI compression of context - just raw .md loading
- No automatic summarization at session end - Claude writes it on trigger
- No sync between machines - it's local files
- No dependency on Obsidian - pure markdown, any editor works
