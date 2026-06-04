# plexus-agent-skills

Traversable agent skills for the Plexus ecosystem. Each skill is a self-contained guide that an AI agent can follow to accomplish a specific task.

**Start here:** [methodology](skills/methodology/) — the high-level spine that ties these skills together: the artifact ladder (goal → milestone → scope → execution → spikes → builds), the loop, and which skill governs each step.

## Why this repo is organized around work planning

Agentic development scales when the **plan** is the artifact, not the agent's memory of a conversation. Once a goal is broken into a dependency DAG of tickets — each ticket carrying its own evidence and contract — the work becomes parallelizable, auditable, and resumable. The agent that writes ticket A is not the agent that implements it; the agent that implements ticket B is not the agent that verifies it. They communicate exclusively through the ticket text.

This is what the process skills in this repo encode. They are not "ways to write good documentation." They are the operating system for multi-agent, multi-session, possibly-multi-day software work. Every other practice — context windows, model selection, autonomy grants, security review — composes on top of a plan that has already paid the cost of being explicit.

The unifying primitive is the **linguistic belief state**: every artifact carries forward both its *value* (the contract, the type, the finding) AND the *evidence* that justifies it. Downstream agents condition on the evidence, not just the value, so when a contract is questioned three steps later the conversation starts from recorded reasoning rather than restarting cold.

In one line: **the human decides; the agent executes; the ticket is the interface.**

## The planning skills

These four skills are the load-bearing surface. Read them in this order if you're new:

| Skill | Role in the planning loop |
|-------|---------------------------|
| [planning](skills/planning/) | Break a goal into a dependency DAG of tickets with explicit inputs, outputs, risks, and parallel paths. Spikes are evidence-gathering steps that update confidence priors on downstream tickets, not binary pass/fail gates. **The plan is a program.** |
| [ticketing](skills/ticketing/) | Write a single ticket that passes the two-stranger test: two agents who have never spoken — one implementing, one verifying — can independently agree on Done using only the ticket text. Each ticket carries an `## Evidence` section downstream tickets condition on. |
| [strong-typing](skills/strong-typing/) | Push the same evidence discipline into the type system. A function taking `FormSlug` doesn't re-validate; the evidence is in the type. Newtypes are the compiler-enforced version of a ticket's contract. |
| [diagramming](skills/diagramming/) | Construct information as high-density representations for human consumption — a diagram compresses language into an image. Diagram over prose; draw a system from many lenses. The reader's bandwidth is the bottleneck. Referenced by planning + ticketing. |
| [autonomous-work](skills/autonomous-work/) | What to do when the user grants you a multi-hour autonomous block. The discipline for working *for* the user (when they aren't reachable) instead of *with* them. Inverse companion to `presence`. |

## Posture skills

Different category — these shape how a conversation runs, not what it produces.

| Skill | When to invoke |
|-------|----------------|
| [presence](skills/presence/) | Substantive collaboration — design, architecture, judgment calls. Bilateral skill (both human and agent read it). Triggers on `/presence`, "let's think through X together," "you make the call," or any explicit invitation to peer collaboration over task execution. |

## Domain skills

| Skill | Description |
|-------|-------------|
| [security-review](skills/security-review/) | Structured security review grouped by SOC2 control families, with multi-trial aggregation and severity calibration. Findings carry an `### Evidence` block with the exploit chain. |
| [forecast](skills/forecast/) | Bayesian Linguistic Forecasting (BLF). Produce a calibrated probability for a binary, dated, observable question, with a structured belief state (probability, confidence, evidence for/against, open questions). |
| [create-plexus-backend](skills/create-plexus-backend/) | Scaffold a new Plexus RPC backend from `templates/` by token substitution. |

## Repo-specific skills (pointers)

These skills live in their respective repos. The entry point is documented here for discoverability.

| Skill | Repo | Entry point | Description |
|-------|------|-------------|-------------|
| Synapse CLI | `synapse/` | `synapse/docs/SYNAPSE.md` | Schema-driven CLI for Plexus RPC — navigation, params, templates |
| Plexus Locus | `plexus-locus/` | `~/CLAUDE.md` → "Plexus Locus" section | Terminal orchestration (tmux/zellij) over Plexus RPC |
| Hyperforge | `hyperforge/` | `~/CLAUDE.md` → "Hyperforge" section | Multi-forge repository management |
| Vox | External: `juggernautlabs/vox/` | `~/CLAUDE.md` → "Vox Development Workflow" section | Live audio transcription pipeline |

## How a planned epic actually flows

1. **Goal in.** The user describes a multi-step outcome.
2. **Apply `planning`.** Decompose into a DAG. Identify roots, leaves, parallel branches. Mark uncertain tickets `confidence: low` and add a spike to their `blocked_by`.
3. **Write each ticket via `ticketing`.** Pin the contract; record the `## Evidence` for *why* the contract has that shape. All tickets start at `status: Pending`.
4. **Human reviews and promotes.** Only the user flips `Pending → Ready`. This is the binding decision boundary; an agent that promotes its own tickets has violated the contract.
5. **Spikes run first** if any low-confidence tickets exist. Spike outputs are structured evidence that updates the unlocked ticket's confidence prior — and may force contract revision before implementation begins.
6. **Implementation runs in parallel** wherever the DAG allows. File-write boundaries are the concurrency unit; tickets that touch disjoint files can run simultaneously.
7. **Each ticket reaches `Complete` only when the integration gate is green** — affected workspace builds and tests pass end-to-end. A `Complete` ticket whose workspace is red is a contract violation.
8. **Calibrate at epic close.** Compare predicted confidence to actual outcome. Adjust priors for the next epic. This is the feedback loop that makes the planner less wrong over time.

Skipping any step shifts cost from one place to another, never eliminates it. Skipping evidence in tickets means downstream agents re-derive reasoning from cold. Skipping spikes means contracts get rewritten mid-implementation. Skipping calibration means the planner stays uncalibrated forever.

## Operating principles

- **The ticket is the prompt.** Ticket quality is the binding constraint on output quality.
- **Documentation is the instruction set.** Context in a human's head but not in a file is a single point of failure for agents.
- **Plans are programs.** Tickets are functions with inputs and outputs. Risks are error conditions. Spikes are error handling that produce evidence, not just status codes.
- **Pending until approved.** Agents write at `status: Pending`. Only humans promote to `Ready`. Implementation must not begin on Pending.
- **Parallelism is the default.** Fan the DAG out. Serial chains are bottlenecks.
- **Verification must be mechanical.** `cargo test` scales. "Does this look right?" doesn't.
- **Calibrate the loop.** At epic close, compare predicted confidence to actual outcome and adjust priors.

The full methodology is documented in `~/CLAUDE.md` → "Methodology — Agentic Development" section.

## Structure

```
skills/
  <skill-name>/
    SKILL.md          # The skill document (entry point, with YAML frontmatter)
    BULK_OPS.md       # Optional: appendix files referenced by the skill
    templates/        # Optional: file templates referenced by the skill
```

## Registering a skill with Claude Code

Each `SKILL.md` has YAML frontmatter (`name:` + `description:`) so it can be registered as a Claude Code Agent Skill. To install one user-wide:

```bash
ln -s "$(pwd)/skills/<skill-name>" ~/.claude/skills/<skill-name>
```

Then Claude Code discovers the skill and surfaces it via the `/<skill-name>` slash command.

## Usage

Point an agent at `skills/<skill-name>/SKILL.md` and it will have everything it needs to execute the task.

For process skills, the agent applies the skill to the current project context. These skills reference `~/CLAUDE.md` for the full convention but are self-contained enough to use standalone.
