# Eval Framework Decision — Discussion Draft

## Background

Sync recap (2026-06-10) agreed:
- Reuse steward's fixture format (step-config.json / report.md / expected.json)
- Runner maintained separately in the Airflow repo
- v1 scope: single-turn, single-model, with-skill vs without-skill

Jarek has since clarified that steward fixture compatibility is not
a hard requirement. This simplifies the framework choice.

I ran a spike testing promptfoo on 3 cases with a 7-arm matrix.
Two findings emerged: one about AGENTS.md, one about case quality.

## The real finding: fixing line 26 alone is not enough

7 arms, 3 cases. Full matrix:

| Arm | Case 1 (contradiction) | Case 2 (Helm) | Case 3 (mypy) |
|-----|:---:|:---:|:---:|
| baseline | FAIL | PASS | FAIL |
| skill-only | PASS | PASS | PASS |
| agents-current | FAIL | PASS | PASS |
| **agents-proposed** | **FAIL** | PASS | PASS |
| skill+agents-current | PASS | PASS | PASS |
| skill+agents-proposed | PASS | PASS | PASS |
| real-env | PASS | PASS | PASS |

`agents-proposed` fixes line 26 ("never run pytest on host" →
"use uv for targeted tests, fall back to breeze"). **It still
fails case-1.** The model chose breeze anyway.

Hypothesis: the 470-line AGENTS.md contains enough breeze-related
commands and context that the model develops a breeze-preference
bias, regardless of one line change. This needs investigation
with more cases — which is exactly what the eval harness is for.

Other findings:
1. Skill is effective: baseline 1/3 → skill-only 3/3
2. Skill overrides the AGENTS.md contradiction when both are present
3. Real-env confirms skill is loaded (matches skill+agents results)

## The bottleneck is cases, not the runner

Of 21 eval runs, 5 arms scored 3/3 — the cases are too easy for
most configurations. The current 3 cases are not discriminating
enough to surface subtle regressions.

The framework should be evaluated on **how efficiently it lets me
expand cases**. Each new case in promptfoo automatically runs
across all arms — one YAML block produces 6 data points.

## What promptfoo gives a developer

A developer modifies SKILL.md or AGENTS.md and asks: "did this
make things better or worse?" Without tooling, they open Claude,
paste a prompt, eyeball the response, open another session without
the change, eyeball again — three repeats and it's chaos.

promptfoo turns that into one command.

### Layer 1: One command to answer "did my change help?"

- **Side-by-side comparison** — before vs after, with-skill vs
  without-skill, one run, one table. Not N manual runs stitched
  together.
- **`--repeat 3`** — agent output is nondeterministic. A single
  "pass" could be luck. Repeat is a built-in flag, not a DIY loop.
- **`skill-used` assertion** — distinguishes "got the right answer
  because the skill guided it" from "got the right answer by
  coincidence." Without this signal, a developer might think their
  skill change worked when the agent never even read it.

### Layer 2: Making "correct" something you can declare

The developer has a standard for what the agent should do — but
that standard usually lives only in their head. Assertions force
it into code:

```yaml
- type: javascript
  value: 'output.runner === "uv"'
- type: skill-used
  value: airflow-contribution
```

Once declared:
- The standard is **executable and repeatable** — anyone modifying
  guidance reruns the same assertions.
- **JSON schema output** (`output_format`) — clean structured
  results, no sed/grep to strip markdown fences.
- **Weighted scoring** — "used the right tool" and "final answer
  correct" score independently, one case gives two dimensions.

This is unit-test discipline for agent guidance. You wouldn't
change code without running pytest; why change guidance without
running eval?

### Layer 3: Regression protection for guidance iteration

Without eval, every AGENTS.md/skill change is blind — fixing A
might silently break B. With a case suite:

- **Regression baseline** — today's pass rate becomes the bar.
  Every future change is asked "better or worse than status quo?"
- **History tracking** — pass rates over time as guidance evolves,
  via web UI.

The developer cost is minimal:
- `npx` — no install, no binary downloads, no package.json changes.
- Declarative YAML — cases are configuration, not code.
- Local-first, open-source — a dev tool, not infrastructure.

## Context degradation: what promptfoo can and cannot do

### Can do: context-as-variable (rough proxy, v1.5)

Treat context length as a test dimension. Pre-build filler texts
of different sizes, prepend them to the probe:

```yaml
tests:
  - vars: { filler: "",                            probe: "...", level: "0k" }
  - vars: { filler: "{{file://fillers/15k.txt}}", probe: "...", level: "15k" }
  - vars: { filler: "{{file://fillers/40k.txt}}", probe: "...", level: "40k" }
```

With `--repeat 5`, this produces "pass rate vs context length" —
a rough degradation curve. Enough to answer "does my skill hold
up in long context, and where does it start to break?"

**Honest limitation:** this simulates degradation as "different
lengths in a single prompt," not "same session accumulating over
multiple turns." Real Claude Code sessions accumulate model-
generated tokens, tool results, and conversation structure — a
pre-built long prompt misses those dynamics. This curve is a
proxy for degradation, not degradation itself.

### Cannot do: real session accumulation (v2, self-built)

True degradation testing requires:
- N turns of real multi-step work (debug, edit, read logs) in one
  session, context accumulates naturally.
- Probe inserted at turn N, checking if the agent still follows
  skill rules.
- Vary N (turn 3/8/15), plot pass rate vs turn depth.

promptfoo's multi-turn support is designed for pipeline correctness,
not "inject a controlled probe at arbitrary session depth and
measure decay." This requires a custom Agent SDK harness.

### Roadmap

| Phase | Tests | Tool |
|-------|-------|------|
| v1 (now) | with/without skill, single-turn | promptfoo |
| v1.5 (after v1 stabilizes) | pass rate vs context length, rough proxy | promptfoo + context-as-vars |
| v2 (if v1.5 shows signal) | pass rate vs session turn depth, reload effect | custom Agent SDK harness |

v1.5 is cheap validation: if even the rough proxy shows no
degradation, v2 may not be needed. If it does, the data justifies
the investment. Either way, v1 must stabilize first — the probe
itself needs to be reliable at 0k context before testing at 40k.

## Architecture

```
dev/skill-evals/
  eval.sh                  ← assembles temp working dirs, runs promptfoo
  promptfooconfig.yaml     ← 6 provider arms + cases + assertions
```

`eval.sh` dynamically builds working directories at eval time:
- SKILL.md copied from repo's single source
- AGENTS.md read from repo (never a stale copy)
- Proposed AGENTS.md generated via `sed` (never stored)
- Temp dirs cleaned up on exit

This preserves Agent SDK provider features (`skill-used`,
`output_format`) while avoiding fixture drift.

## No architectural dependency on promptfoo

promptfoo is owned by OpenAI (acquired 2026-03). For an ASF
project, building architectural dependency on a single commercial
entity's tool is a concern — regardless of current license.

This eval harness deliberately avoids that dependency:

- **promptfoo is an execution convenience layer, not an
  architectural choice.** The investment is in test cases (YAML
  with domain knowledge about Airflow's command routing), not in
  runner infrastructure.
- **No CI integration.** Runs manually, on demand, as a dev-time
  decision tool. Not in any build path, release gate, or workflow.
- **Cases are portable.** A 50-line Python script can read the
  same YAML and pipe to `claude -p` via `subprocess`. Migration
  cost: ~1 day.
- **MIT license.** If the project changes direction, the last
  open version can be forked.

## Impact assessment

| Concern | Severity | Mitigation |
|---------|:--------:|------------|
| OpenAI ownership — ASF optics | Medium | No architectural dependency; execution-only; not in CI or build path |
| Version not pinned | Low | `npx promptfoo@0.120.19` to lock |
| Not in package.json / pyproject.toml | Low | Dev-only tool, documented in README |
| LLM call cost | Low | Manual-only, on demand before changes |

## Proposal

Use promptfoo with the Claude Agent SDK provider as the eval
runner for Airflow skill-evals:

- Config: `dev/skill-evals/promptfooconfig.yaml`
- Setup + run: `dev/skill-evals/eval.sh`
- Run manually, not in CI — a dev-time decision tool

### Use cases

1. **Before modifying SKILL.md** — run eval to confirm current
   pass rates, modify, re-run, compare
2. **Before modifying AGENTS.md** — agents-current vs agents-proposed
   arm detects regressions
3. **Skill effectiveness measurement** — skill-only vs baseline
   quantifies the skill's contribution

### Immediate next step

Expand from 3 to 10+ discriminating cases — provider tests with
system dep failures, uv fallback scenarios, cross-package changes,
breeze suite selection. The cases are the bottleneck, not the
framework.

## Open questions for discussion

- Is the Node.js dependency acceptable for a dev-only eval tool?
- Does the `anthropic:claude-agent-sdk` provider require API key
  access for GSoC contributors, or is there a project key available?
