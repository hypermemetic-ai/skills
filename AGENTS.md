# skills

Generic [Kimi Code](https://www.kimi.com/code) skill authoring guidelines.

> **Note:** The abstract methodology that used to live here has moved to [`ought`](https://github.com/hypermemetic-ai/ought). This repo now contains only Kimi-Code-specific mechanics.

## Skill file convention

- One skill = one `SKILL.md` under `skills/<skill-name>/`. Supporting files live in the same directory.
- Frontmatter has at least `name` and `description`.
- Body should be clear, concise, and focused on Kimi Code behavior: how skills register as `/commands`, how to invoke them, how to package plugins, etc.
- Keep skills methodology-agnostic. Methodology content belongs in `ought`.

## Useful references

- Agent Skills: https://www.kimi.com/code/docs/en/kimi-code-cli/customization/skills.html
- Slash commands: https://www.kimi.com/code/docs/en/kimi-code-cli/reference/slash-commands.html
- Plugins: https://www.kimi.com/code/docs/en/kimi-code-cli/customization/plugins.html

## Git workflow

Keep `main` clean and its history linear:

- Land every change via a feature branch + PR — don't push to `main` directly.
- Keep history linear (rebase, don't merge-commit feature branches). Pushing to feature branches is unrestricted.
