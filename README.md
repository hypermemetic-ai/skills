# plexus-agent-skills

Traversable agent skills for the Plexus ecosystem. Each skill is a self-contained guide that an AI agent can follow to accomplish a specific task.

## Skills

### Implementation skills

| Skill | Description |
|-------|-------------|
| [create-plexus-backend](skills/create-plexus-backend/) | Scaffold and implement a new Plexus RPC backend server |

### Process skills

| Skill | Description |
|-------|-------------|
| [ticketing](skills/ticketing/) | Write a TDD ticket that passes the two-stranger test |
| [planning](skills/planning/) | Break a goal into a dependency DAG of tickets with inputs, outputs, and risks |
| [security-review](skills/security-review/) | Structured security review grouped by SOC2 control families |
| [strong-typing](skills/strong-typing/) | Introduce newtypes for domain concepts so the compiler catches misuse |

### Repo-specific skills (pointers)

These skills live in their respective repos. The entry point is documented here for discoverability.

| Skill | Repo | Entry point | Description |
|-------|------|-------------|-------------|
| Synapse CLI | `synapse/` | `synapse/docs/SYNAPSE.md` | Schema-driven CLI for Plexus RPC — navigation, params, templates |
| Plexus Locus | `plexus-locus/` | `~/CLAUDE.md` → "Plexus Locus" section | Terminal orchestration (tmux/zellij) over Plexus RPC |
| Hyperforge | `hyperforge/` | `~/CLAUDE.md` → "Hyperforge" section | Multi-forge repository management |
| Vox | External: `juggernautlabs/vox/` | `~/CLAUDE.md` → "Vox Development Workflow" section | Live audio transcription pipeline |
| FormVeritas auth model | `FormVeritasV2/` | `FormVeritasV2/CLAUDE.md` → "Roles and Tenant Scoping" | TenantScope, ValidUser, auth chain |

## Methodology

The process skills encode a methodology for agentic development — where an AI agent is the primary implementor and the human is the decision-maker. The full methodology is documented in `~/CLAUDE.md` → "Methodology — Agentic Development" section. Key principles:

- **The ticket is the prompt.** Ticket quality is the binding constraint on output quality.
- **Documentation is the instruction set.** Every piece of context that's in a human's head but not in a file is a single point of failure for agents.
- **Plans are programs.** Tickets are functions with inputs and outputs. Risks are error conditions. Spikes are error handling.
- **Pending until approved.** Agents write tickets at `status: Pending`. Only humans promote to `Ready`. Implementation must not begin on Pending tickets.

## Structure

```
skills/
  <skill-name>/
    SKILL.md          # The skill document (entry point)
    templates/        # Optional: file templates referenced by the skill
```

## Usage

Point an agent at `skills/<skill-name>/SKILL.md` and it will have everything it needs to execute the task.

For process skills (ticketing, planning, security-review, strong-typing), the agent applies the skill to the current project context. These skills reference `~/CLAUDE.md` for the full convention but are self-contained enough to use standalone.
