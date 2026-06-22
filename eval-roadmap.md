# Skill-Eval Harness Roadmap

Status as of 2026-06-14.

## Current state

- `dev/skill-evals/eval.sh` — quick mode (2 arms) / full mode (5 arms)
- `dev/skill-evals/cases/command-routing.yaml` — 3 cases
- Claude Agent SDK provider with `skill-used` and `output_format`
- Prek hook reminder on AGENTS.md / SKILL.md changes
- Branch: `feat/skill-eval-harness`

## Phase 1: Case expansion (priority, 2-3 days)

The bottleneck is case quality, not the framework. Expand from
3 to 10+ discriminating cases.

### Cases to add

**command-routing.yaml (expand):**

| Case | Scenario | Expected |
|------|----------|----------|
| uv fallback | uv fails with system dep error | breeze |
| provider single | run amazon provider tests | breeze (`Providers[amazon]`) |
| provider multi | run amazon + google providers | breeze (`Providers[amazon,google]`) |
| scripts tests | run scripts/tests/ | uv (`--project scripts`) |
| task-sdk tests | run task-sdk tests | breeze (`task-sdk-tests`) |
| db tests parallel | run only DB tests in parallel | breeze (`--run-db-tests-only`) |

**static-checks.yaml (new):**

| Case | Scenario | Expected |
|------|----------|----------|
| ruff lint | lint changed files | prek (`ruff --from-ref`) |
| ruff format single file | format one .py file | uv (`uv run ruff format`) |
| mypy provider | type-check a provider | breeze (`breeze run mypy`) |
| fast static gate | run pre-push checks | prek (`--stage pre-commit`) |

**edge-cases.yaml (new):**

| Case | Scenario | Expected |
|------|----------|----------|
| selective-checks | determine which tests to run | breeze (`selective-checks`) |
| airflow CLI | run `airflow dags list` | breeze (`breeze run airflow`) |

## Phase 2: --model flag (1 hour)

Add `--model` parameter to `eval.sh`:

```bash
./eval.sh --model claude-haiku-4-5-20251001   # cheap, fast iteration
./eval.sh                                      # default: sonnet
./eval.sh --model claude-opus-4-6              # most accurate
```

Arm count stays at 2, only the underlying model changes.

## Phase 3: Report output (half day)

Add `--report` flag to generate markdown for PR descriptions:

```bash
./eval.sh --report > eval-results.md
```

Output:

```markdown
## Eval results (vs main)

Changes: AGENTS.md modified

| Case | before | after |
|------|--------|-------|
| Core unit test (uv) | PASS | PASS |
| Helm test (breeze) | PASS | PASS |
| mypy (prek) | FAIL | PASS |

Pass rate: 2/3 → 3/3
```

## Phase 4: Context degradation proxy (1-2 days)

Prerequisite: Phase 1 stable.

Use context-as-variable to rough-test whether skills hold up
in long context:

```
cases/
  context-degradation.yaml
fillers/
  15k.txt
  40k.txt
  80k.txt
```

```yaml
- description: "Core unit test @ 0k context"
  vars: { filler: "", request: "Run test_taskinstance.py..." }
- description: "Core unit test @ 40k context"
  vars: { filler: "{{file://fillers/40k.txt}}", request: "..." }
```

Run with `--repeat 5`, plot pass rate vs context length.

**Limitation:** this is a proxy (single-turn with pre-built filler),
not real session accumulation. It answers "does the skill survive
long prompts?" not "does the skill survive 15 turns of real work."

## Phase 5: Routing boundary testing (1 day)

Prerequisite: Phase 1 stable.

Test that the skill does NOT trigger on unrelated tasks:

```yaml
- description: "Code review should NOT trigger contribution skill"
  vars:
    request: "Review this PR for code quality issues."
  assert:
    - type: not-skill-used
      value: airflow-contribution
```

Prevents over-triggering.

## Phase 6: Multi-turn session testing (3-5 days)

Prerequisite: Phase 4 shows degradation signal.

Self-built harness using `@anthropic-ai/claude-agent-sdk` directly.
Controls a multi-turn conversation: accumulate context over N turns
of real work, insert probe at fixed turn depths, measure pass rate
decay.

```
Turn 1-5:  real tasks (read code, review diff, explain module)
Turn 6:    probe ("run test_taskinstance.py") → record pass/fail
Turn 7-10: more real tasks
Turn 11:   same probe → record pass/fail
```

Plot pass rate vs turn depth. Compare arms:
- Arm A: skill present from turn 1
- Arm B: skill reloaded at each probe turn
- Arm C: no skill (control)

**Only build if Phase 4 confirms degradation exists.**

## Decision points

- After Phase 1: are cases discriminating enough? Do any arms
  show unexpected failures?
- After Phase 4: is there measurable degradation? If yes → Phase 6.
  If no → stop, the skill is robust enough.
- After Phase 5: does the skill over-trigger? If yes → tighten
  skill description.
