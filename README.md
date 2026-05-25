# super-mem

Persistent cross-session context loading + daily work logging for Claude Code.

## What it does

Gives any project a "brain" - a structured markdown knowledge base + daily work log that loads into every Claude Code session. Solves the "Claude starts cold" problem.

## How it works

1. **Setup** creates a `docs/brain/` directory with structured markdown templates and registers a SessionStart hook
2. **Every session**, the hook collects all brain files + Claude memories + daily logs, writes them to disk
3. **Say "session start"** and Claude reads the full context file (~50K tokens typical)
4. **Say "session end"** and Claude logs what happened to today's daily file
5. **Next session**, the new Claude picks up right where you left off

## Install

```
/install super-mem
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
