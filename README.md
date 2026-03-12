# plexus-agent-skills

Traversable agent skills for the Plexus ecosystem. Each skill is a self-contained guide that an AI agent can follow to accomplish a specific task.

## Skills

| Skill | Description |
|-------|-------------|
| [create-plexus-backend](skills/create-plexus-backend/) | Scaffold and implement a new Plexus RPC backend server |

## Structure

```
skills/
  <skill-name>/
    SKILL.md          # The skill document (entry point)
    templates/        # Optional: file templates referenced by the skill
```

## Usage

Point an agent at `skills/<skill-name>/SKILL.md` and it will have everything it needs to execute the task.
