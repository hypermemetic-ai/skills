# skills

Generic [Kimi Code](https://www.kimi.com/code) skills, slash commands, and plugin guidance.

> **Where did the methodology go?** The project-specific methodology skills used to live here. They now live in [`ought`](https://github.com/hypermemetic-ai/ought), which houses the abstract methodology entirely. This repo is reserved for Kimi-Code-specific mechanics that are independent of any particular methodology.

## Layout

```
skills/
├── README.md          # this file
├── AGENTS.md          # conventions for skills in this repo
└── kimi/              # how to write Kimi skills, /commands, and plugins
    └── README.md
```

## Adding a generic skill

1. Create a directory under `skills/<name>/`.
2. Add a `SKILL.md` with YAML frontmatter (`name`, `description`) and Markdown body.
3. Keep it Kimi-Code-specific and methodology-agnostic.

For the full skill format, locations, invocation, and plugin packaging, see [`kimi/README.md`](kimi/README.md) and the linked official Kimi docs.
