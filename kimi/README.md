# Kimi Code skills, slash commands, and plugins

This directory once held project-specific methodology skills. That methodology now lives in [`ought`](https://github.com/hypermemetic-ai/ought). This repo is now reserved for **generic, Kimi-Code-specific guidance** — how to write skills, register `/commands`, and package plugins.

## Quick links to Kimi's official docs

| Topic | Kimi doc |
|---|---|
| Agent Skills (SKILL.md format, locations, invocation) | https://www.kimi.com/code/docs/en/kimi-code-cli/customization/skills.html |
| Slash commands reference | https://www.kimi.com/code/docs/en/kimi-code-cli/reference/slash-commands.html |
| Plugins (packaging skills + MCP servers) | https://www.kimi.com/code/docs/en/kimi-code-cli/customization/plugins.html |
| Using skills in Kimi Code | https://www.kimi.com/help/agent/use-skills-in-code |
| What are skills? | https://www.kimi.com/help/agent/what-are-skills |

## The short version

A **Skill** is a Markdown file with YAML frontmatter. When Kimi Code discovers it, the skill can be invoked as a slash command.

### File format

Directory form (recommended):

```
~/.kimi-code/skills/my-skill/SKILL.md
```

```markdown
---
name: my-skill
description: One-line summary of what this skill does and when to use it.
type: prompt
whenToUse: When the user asks for X
arguments:
  - target
---

Your skill instructions here. Use `$target` to reference the argument.
```

### Skill locations (precedence: Project > User > Extra > Built-in)

| Scope | Paths |
|---|---|
| Project | `.kimi-code/skills/`, `.agents/skills/` (nearest `.git` root) |
| User | `~/.kimi-code/skills/`, `~/.agents/skills/` |
| Extra | declared in `config.toml` under `extra_skill_dirs` |

### Slash commands

- `/skill:my-skill [args]` — explicit invocation
- `/my-skill [args]` — shorthand if the name does not collide with a built-in command
- Sub-skills appear as `/parent.child`

### Plugins

For distribution, package skills in a plugin with a `kimi.plugin.json` manifest. Plugins can also declare `mcpServers` for real tool capabilities.

See the official links above for the full spec, argument placeholders (`$0`, `$ARGUMENTS`, `${KIMI_SKILL_DIR}`), `type: flow`, and security model.
