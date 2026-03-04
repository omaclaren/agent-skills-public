# Agent Skills (Public)

Open-source agent skills following the [Agent Skills](https://agentskills.io) specification. Compatible with any agent that supports the standard — Claude Code, Cursor, Gemini CLI, OpenCode, Codex, pi, and others.

## Skills

| Skill | Description |
|-------|-------------|
| See `skills/` directory | Each skill has its own `SKILL.md` with full details |

## Usage

### Manual install

Clone and copy any skill directory to your agent's skills location:

```bash
# Example for ~/.agents/skills/ (cross-harness)
cp -r skills/guide-mode ~/.agents/skills/

# Example for ~/.claude/skills/ (Claude Code)
cp -r skills/guide-mode ~/.claude/skills/
```

### As a Claude Code plugin marketplace

```
/plugin marketplace add <github-user>/agent-skills-public
```

## Specification

All skills follow the [Agent Skills specification](https://agentskills.io/specification). Each skill is a directory containing a `SKILL.md` with YAML frontmatter (`name`, `description`) and Markdown instructions.

## License

Apache-2.0
