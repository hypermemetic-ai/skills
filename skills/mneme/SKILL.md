---
name: mneme
description: Drive the [mneme](https://github.com/hypermemetic/mneme) forecasting substrate — a working implementation of the Bayesian Linguistic Forecaster from [Murphy 2026 (arXiv:2604.18576)](https://arxiv.org/abs/2604.18576). Use when the user asks to forecast a binary event, record a resolved outcome, run the ForecastBench benchmark, watch live Manifold markets, predict the outcome of a software design decision, or interrogate the substrate's calibration over time. Distinct from the `forecast` skill, which is the trial-side prompt loaded INTO the substrate's reasoners; this skill is the OPERATOR-side prompt that drives the substrate as a tool.
---

# Skill: Operate the mneme forecasting substrate

The mneme substrate ([github.com/hypermemetic/mneme](https://github.com/hypermemetic/mneme)) is a Plexus RPC server implementing the Bayesian Linguistic Forecaster (BLF) algorithm from Murphy 2026. It produces calibrated probabilities + structured rationale for binary forecasting questions, and the same machinery is used to forecast the consequences of software design decisions.

This skill drives the substrate. When invoked you have shell access (Bash) and `synapse` on PATH; the substrate is expected to be running in a container on `ws://localhost:4456`.

## When to invoke

- The user asks for a probability on a binary, dated, observable question. ("Will X happen by Y?")
- The user wants to record an outcome they now know about a forecast they (or you) previously fired.
- The user wants to run the ForecastBench benchmark end-to-end and see Brier Index numbers.
- The user wants to watch live prediction-market questions — fire forecasts, log paired data, sweep resolutions.
- The user is making a software design decision and wants to record a prediction about how it will pan out.
- The user wants to inspect what the substrate has been doing — resolved observations, calibration drift, recent forecasts.
- The user uses the words "calibrate," "Brier," "BLF," "ForecastBench," "Platt," or "predict the next bench."

If the user is asking a **conceptual** question about forecasting that doesn't need the running system (e.g. "what's a Brier score?"), answer directly without invoking this skill.

## Prerequisites

**1. Substrate must be running.** Check with:

```bash
docker ps --filter name=mneme --format '{{.Names}} {{.Status}}'
```

If empty: tell the user the substrate isn't running and propose:

```bash
cd ~/dev/controlflow/hypermemetic/mneme-substrate
make build && make run     # ← drops them into container shell
```

`make run` auto-runs `claude /login` if no OAuth token is found. The substrate persists state under `./programs/` and `./.plexus-state/` (bind-mounted; survives container restarts).

**2. `synapse` must be on PATH.** If not: tell the user to install [synapse](https://github.com/hypermemetic/synapse) on their host, or run from inside the container shell where the substrate is reachable on `localhost:4456`.

## Operations

### Fire a forecast

The substrate's `forecast.update` is **fire-and-return** — it kicks the work into the background and returns a `program_id` immediately. Use `programs wait` to block until the answer lands.

```bash
synapse -P 4456 substrate forecast update \
    --program-id <stable-question-id> \
    --new-evidence "<question text + any context>" \
    --trials 3 \
    --iterative-max-steps 5
```

Output looks like:
```
prior:
  belief_schema_version: 0.3.0
  ...
program_id: 7fbbf382-accd-4926-8a2b-b9e6d68df59d
type: started
```

**Capture the `program_id`** from synapse's output (it's printed verbatim). Then:

```bash
synapse -P 4456 substrate programs wait --program-id 7fbbf382-...
```

This streams `progress` events on each status change and ends with `completed` carrying the artifact:

```
type: completed
program_id: 7fbbf382-...
artifact:
  probability: 0.140
  raw_probability: 0.180
  evidence_for: [...]
  evidence_against: [...]
  open_questions: [...]
  summary: "FOR: ... AGAINST: ..."
  n_trials: 3
  confidence: multi-trial
```

**Render the result for the user clearly.** Don't dump the YAML — say something like:

> The system predicts **14.0%** probability (raw 18%, softened by Platt calibration). Based on:
> - **For YES**: [top 2-3 evidence_for claims]
> - **Against**: [top 2-3 evidence_against claims]
> - **Open questions** (the system flagged things it couldn't determine): [list]
>
> Recorded as program `7fbbf382-...`; resolve it with `mneme.skill > forecast.resolve` when the outcome is known.

### Resolve an outcome

When the user knows what actually happened:

```bash
synapse -P 4456 substrate forecast resolve \
    --program-id <id> \
    --actual true   # or false
```

This appends a `(predicted, actual)` row to `programs/_calibration/history.jsonl` and refits the Platt parameters once ≥10 resolutions exist. The next forecast benefits from the tighter fit.

### Run the ForecastBench benchmark

```bash
# Vendor data first (CC BY-SA 4.0); skip if already there
mkdir -p ~/dev/controlflow/hypermemetic/mneme-substrate/programs/_benchmarks/forecastbench
cd ~/dev/controlflow/hypermemetic/mneme-substrate
curl -sL -o programs/_benchmarks/forecastbench/2024-07-21-llm.json \
  https://raw.githubusercontent.com/forecastingresearch/forecastbench-datasets/main/datasets/question_sets/2024-07-21-llm.json
curl -sL -o programs/_benchmarks/forecastbench/2024-07-21_resolution_set.json \
  https://raw.githubusercontent.com/forecastingresearch/forecastbench-datasets/main/datasets/resolution_sets/2024-07-21_resolution_set.json

# n=20 paired vs the prediction-market crowd — ~10 min wall-clock
python3 scripts/forecastbench_live_run.py \
  --question-set programs/_benchmarks/forecastbench/2024-07-21-llm.json \
  --resolution-set programs/_benchmarks/forecastbench/2024-07-21_resolution_set.json \
  --n 20 --concurrency 4 --port 4456 --trials 2 --iterative-max-steps 5 \
  --output programs/_benchmarks/runs/$(date +%Y%m%d-%H%M%S)-mybench/
```

The script reports independent + **paired** Brier Index (mneme vs crowd) with 95% bootstrap CIs and a per-source breakdown. Walk the user through the numbers — the paired delta is the headline.

### Live Manifold marketplace

```bash
cd ~/dev/controlflow/hypermemetic/mneme-substrate
python3 scripts/marketwatch_live.py --max-markets 10 --port 4456    # forecast pass
python3 scripts/marketwatch_resolve.py --port 4456                   # resolution sweep
```

Pairings accumulate in `programs/_marketwatch/pairings.jsonl`. Designed for cron — see `scripts/marketwatch_README.md`. **This is the cleanest data path** because the markets haven't resolved yet at forecast time, so no web-search contamination is possible.

### Forecast a software design decision

Add a `forecast:` block to a ticket's frontmatter:

```yaml
---
id: MY-TICKET-1
forecast:
  hypothesis: "Will <measurable outcome> by <date>?"
  resolution_method: "Run X; check Y; YES if Z."
  deadline: "2026-06-01T00:00:00Z"
---
```

Then:

```bash
cd ~/dev/controlflow/hypermemetic/mneme-substrate
python3 scripts/ticket_forecast.py --plans-dir ../mneme/plans
python3 scripts/ticket_resolve.py --plans-dir ../mneme/plans
```

Predictions append to `<plans>/_predictions.jsonl`. After deadline, resolutions feed `forecast.resolve` so the calibration store grows with **design-judgment data** — meta-evidence about whether the system's design intuitions are reliable.

## Output discipline

When you give a forecast result back to the user:

1. **Lead with the calibrated probability**, then the raw if it differs meaningfully.
2. **Show the top evidence on each side** — 2-3 claims, with weights if available. Skip the claims with weight < 0.3 unless they're the only evidence.
3. **Surface the open questions** — these tell the user what would tighten the answer. Sometimes worth proposing follow-up forecasts on them.
4. **Note the program_id** so the user can resolve later.
5. **State the calibration regime** if it matters: "this is post-Platt with N=114 resolved observations" vs "cold-start, no Platt yet."

When you give bench results:
1. **Paired delta first**, with CI. Independent BIs second.
2. **Per-source breakdown** if the paired sample is mixed (manifold/metaculus/polymarket/infer behave differently).
3. **Honest caveats**: small n means wide CI; ForecastBench resolutions before the model's training cutoff have contamination concerns; web-search-during-iterative-loop can retrieve resolution articles for past events.

## The recursion (this is the load-bearing part)

When the user is making a software design decision — refactor, new feature, architectural choice — **propose recording it as a forecast**. Phrase the decision as a binary, dated, resolvable question:

- "Will this refactor produce a measurable improvement (X metric ≥ Y) by Z date?"
- "Will this experiment land within 2 weeks AND show effect size ≥ 5% on the chosen metric?"
- "Will this dependency replacement reduce wall-clock by ≥20% on bench-005?"

Filing the prediction is one command. Resolving it later is one command. After 30+ resolved design-forecasts the user has actual data on whether the system's design intuitions are trustworthy — meta-evidence that compounds.

## Honesty discipline

Surface, don't hide:
- **Contamination** on benches: ForecastBench questions resolving in the model's training window are not held-out evaluation, they're "lookup" tests.
- **Sample-size limits**: n=20 has 95% CI on mean Brier of [~0.004, ~0.197], translating to ~25 BI of noise band. Don't celebrate small-sample wins.
- **Calibration cold-start**: with <10 resolutions, Platt is identity. The substrate falls back to raw probabilities.
- **Web-search-after-the-fact**: even on post-cutoff questions, the iterative loop can retrieve news articles about resolutions that already happened. The structural defense is BLFX-9 (date-leakage prompt), which isn't built yet.

## Schema reminder (for parsing artifact JSON)

```rust
ForecastState {
    probability: f64,                 // post-Platt (calibrated)
    raw_probability: Option<f64>,     // pre-Platt
    confidence: ForecastConfidence,   // multi-trial | single-pass | low | medium | high
    evidence_for: Vec<EvidenceItem>,  // {claim, source: Option<String>, weight: f64}
    evidence_against: Vec<EvidenceItem>,
    open_questions: Vec<String>,
    summary: String,                  // auto-rendered prose from structured fields
    n_trials: u8,
    prior_used: Option<PriorRef>,
    belief_schema_version: String,    // "0.3.0" current
}
```

EvidenceItem deserializer is **lenient** — accepts either the full object form or a bare string (which normalizes to `{claim: <string>, source: None, weight: 0.5}`). Surfaced because some agents emit evidence as strings; the substrate handles it.

## Out of scope for this skill

- Implementing new substrate-side methods (that's a substrate engineering task; use the `ticketing` skill to write a ticket first).
- Polymarket integration (currently Manifold only — Polymarket is a future ticket).
- Real-money bet placement (the substrate observes; it does not bet).
- Modifying the calibration store schema or Platt math (substrate engineering).

## References

- [Murphy 2026, BLF](https://arxiv.org/abs/2604.18576) — the algorithm
- [mneme repo](https://github.com/hypermemetic/mneme) — concept, tickets, paper-faithfulness story, bench results
- [mneme-substrate repo](https://github.com/hypermemetic/mneme-substrate) — the Rust server + scripts
- Bench result docs: `~/dev/controlflow/hypermemetic/mneme/plans/BLFX/results/`
- Existing forecast trial-side prompt: `~/dev/controlflow/hypermemetic/skills/skills/forecast/SKILL.md` (loaded into trials, NOT this operator skill)
