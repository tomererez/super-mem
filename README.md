# super-mem

Persistent cross-session context loading + daily work logging for Claude Code.

## What it does

Every Claude Code session starts from zero. Claude doesn't know your architecture, your business rules, yesterday's decisions, or what you were working on an hour ago. You end up repeating context, re-explaining constraints, and watching Claude make mistakes you've already corrected.

**super-mem** solves this by giving your project a persistent "brain" - a structured markdown knowledge base + daily work log that loads into every session automatically.

### Three layers of memory

1. **Brain** (`docs/brain/`) - Your project's knowledge base: architecture, business rules, team workflows, deployment process. Structured markdown files you maintain over time. This is the stuff that doesn't change often but Claude needs to know every session.

2. **Auto-memory** (`~/.claude/projects/.../memory/`) - Claude's own memory files. Things Claude learned about your preferences, your codebase patterns, your workflow. super-mem collects these too.

3. **Daily log** (`docs/brain/daily/`) - Session-by-session work log. What was done, what was decided, what's still open. The last 14 days load automatically, so Claude sees recent history and picks up where you (or your teammate) left off.

### Why it works

- **It's just markdown.** No database, no sync service, no AI compression. Plain `.md` files you can read, edit, and version control. Works with any editor.
- **It bypasses the 10KB hook limit.** Claude Code's `additionalContext` is capped at 10KB. super-mem writes the full context to disk (no size limit), then Claude reads it on demand. A typical medium project loads ~45K tokens, under 5% of the 1M context window.
- **Sessions build on each other.** Each "session end" appends a structured summary to the daily log. Next session, that summary is part of the loaded context. Claude sees what happened before and doesn't repeat work or contradict past decisions.
- **It's cheap on tokens.** The brain loads only when you ask for it ("session start"). If you don't need full context for a quick task, just skip it.

Built and validated on a production SaaS codebase (65 brain files + 55 memory files + daily logs = 218KB, ~56K tokens, running since May 2026).

## How it works

1. **Setup** (`/super-mem setup`) creates a `docs/brain/` directory with structured markdown templates and registers a SessionStart hook
2. **Every session**, the hook automatically collects all brain files + Claude memories + daily logs, writes them to a single file on disk
3. **Say "session start"** and Claude reads the full context file (~50K tokens typical)
4. **Say "session end"** and Claude logs a structured summary to today's daily file
5. **Next session**, the new Claude picks up right where you left off

## Install

```
/install-plugin tomererez/super-mem
```

## Quick Start

```
/super-mem setup
```

Answer 5 questions about your project. The skill creates your brain structure and registers the hook automatically.

## Commands

| Command | What it does |
|---------|-------------|
| `/super-mem` | Show status and help |
| `/super-mem setup` | Set up brain for this project |
| `/super-mem expand <module>` | Add a module after setup |

## Triggers

| Phrase | Action |
|--------|--------|
| "session start" | Load full project context |
| "session end" | Log session summary to daily file |
| "end of day" | Session summary + team daily summary |

## Modules

The setup flow asks questions and creates modules based on your answers. You can also add them later with `/super-mem expand <module>`.

| Module | When to use |
|--------|------------|
| `team` | Working with others |
| `workflows` | Git flow, PR rules, conventions |
| `domain` | Complex business logic |
| `concepts` | Entity definitions |
| `operations` | Deploying to production |
| `reference` | External systems, API keys |

## Token Budget

| Size | Typical tokens | % of 1M |
|------|---------------|---------|
| Small (~10 brain files) | ~35K total | 3.5% |
| Medium (~30 brain files) | ~45K total | 4.5% |
| Large (~65 brain files) | ~60K total | 6% |

Target: under 10% of context window.

## License

MIT
