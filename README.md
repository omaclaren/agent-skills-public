# Agent Skills

Some simple agent skills. Follow the [Agent Skills specification](https://agentskills.io).

## Skills

| Skill | Description |
|-------|-------------|
| **guide-mode** | Walkthrough-first collaboration mode — small steps, user sign-off before mutations |
| **critique-skill** | Structured critique of writing or code |
| **annotated-reply-skill** | Copy the last response to clipboard with an annotation header, ready to paste back and mark up |
| **preview-browser-skill** | Render markdown as HTML and open in the browser |

## Install

Copy any skill folder to your agent's skills directory:

```bash
cp -r skills/guide-mode ~/.agents/skills/
```

Or for Claude Code specifically:

```bash
cp -r skills/guide-mode ~/.claude/skills/
```

Each skill is just a folder with a `SKILL.md` — works with any compatible agent.

For Claude Code, you can also install via the plugin marketplace:

```
/plugin marketplace add omaclaren/agent-skills-public
```

## License

MIT
