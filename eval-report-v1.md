# Eval Framework Spike — Report

Date: 2026-06-13

## Goal

Evaluate whether an off-the-shelf eval framework can replace a custom
runner for the Airflow skill-eval harness (GSoC Task 4), and produce
initial evidence for skill effectiveness and AGENTS.md quality.

## Frameworks Tested

| Framework | Tested? | Notes |
|-----------|:-------:|-------|
| steward runner | Yes | Copied from airflow-steward, fixture-based, stdin pipe |
| promptfoo | Yes | YAML config, `exec:` provider, built-in assertions |
| Inspect AI | No | Requires API key (not available with Max subscription) |

## Key Finding: `cd /tmp` Solves Isolation

`claude -p --bare` does not work with Max OAuth login. However,
running `claude -p` from `/tmp` prevents auto-discovery of
CLAUDE.md / AGENTS.md — achieving the same isolation without `--bare`.

This enables true with/without comparisons at zero cost.

## Test Matrix: 4-Layer, 7-Arm Design

Each arm controls what the model sees:

| Arm | Skill | AGENTS.md | CLAUDE.md | Purpose |
|-----|:-----:|:---------:|:---------:|---------|
| 1-baseline | — | — | — | What the model knows without any guidance |
| 1-skill-only | yes | — | — | Does the skill alone provide correct routing? |
| 2-agents-current | — | current | — | How does the existing AGENTS.md perform? |
| 2-agents-proposed | — | proposed | — | Does fixing line 26 resolve the contradiction? |
| 3-skill+agents-current | yes | current | — | Does the skill survive AGENTS.md interference? |
| 3-skill+agents-proposed | yes | proposed | — | Intended production state |
| 4-real-env | auto | auto | auto | Does the skill actually load in practice? |

### Test Cases

| Case | Scenario | Expected Runner | Discriminating? |
|------|----------|:---------------:|:---------------:|
| case-1 | Run test_taskinstance.py (AGENTS.md line 26 vs line 35) | uv | Yes |
| case-2 | Run Helm chart tests | breeze | No |
| case-3 | Run mypy on airflow-core | prek | Partial |

## Results

### Pass/Fail Matrix

| Arm | Case 1 (contradiction) | Case 2 (Helm) | Case 3 (mypy) | Pass Rate |
|-----|:----------------------:|:-------------:|:-------------:|:---------:|
| 1-baseline | FAIL | PASS | FAIL | 1/3 |
| 1-skill-only | PASS | PASS | PASS | 3/3 |
| 2-agents-current | FAIL | PASS | PASS | 2/3 |
| 2-agents-proposed | FAIL | PASS | PASS | 2/3 |
| 3-skill+agents-current | PASS | PASS | PASS | 3/3 |
| 3-skill+agents-proposed | PASS | PASS | PASS | 3/3 |
| 4-real-env | PASS | PASS | PASS | 3/3 |

### Interpretation

**1. Skill is effective (Layer 1).**
Baseline scores 1/3. With skill: 3/3. Delta: +0.67.
The skill provides correct routing guidance that the model lacks
on its own. Notably, without the skill the model:
- Chose `breeze` for case-1 (influenced by general knowledge
  that db_test implies database = container)
- Guessed `mypy-airflow` instead of the correct `mypy-airflow-core`
  for case-3 (doesn't know the exact prek hook name)

**2. Fixing line 26 alone is insufficient (Layer 2).**
`agents-proposed` still fails case-1. The proposed fix changed
line 26 from "Never run pytest on the host" to "Use uv for targeted
tests, fall back to breeze." But the model still chose breeze.

Hypothesis: the 470-line AGENTS.md contains enough breeze-related
commands and context that the model develops a breeze-preference
bias, regardless of one line. This needs further investigation
with more cases.

**3. Skill overrides AGENTS.md contradiction (Layer 3).**
When both skill and AGENTS.md are present, the skill's explicit
routing rules take priority. Even with the contradictory
line 26, `skill+agents-current` passes all 3 cases.

This means the skill is not just additive — it actively corrects
the AGENTS.md contradiction in practice.

**4. Real environment confirms skill discovery (Layer 4).**
`4-real-env` matches `3-skill+agents` results (3/3), confirming
that when `claude -p` runs from the repo root, it loads the same
context as our manually-piped skill + AGENTS.md combination.

## Framework Comparison

### steward runner

| Dimension | Assessment |
|-----------|------------|
| Setup to first case | Fast — copy runner.py + write fixtures |
| Fixture format | Familiar (report.md + expected.json per case dir) |
| CLI integration | Native stdin pipe, zero issues |
| Two-arm comparison | Must run twice, compare manually |
| JSON handling | Built-in 3-layer extraction (pure → fence → brace) |
| Grading | 3-tier (exact / Haiku judge / structural) |
| Scaling | Tag-based filtering, no built-in history |

**Strength:** JSON extraction resilience, Haiku judge for prose fields.
**Weakness:** No built-in multi-arm comparison; no eval history.

### promptfoo

| Dimension | Assessment |
|-----------|------------|
| Setup to first case | Medium — YAML + shell scripts + debug code fence issue |
| Fixture format | YAML (inline or `file://cases/*.yaml` for scale) |
| CLI integration | `exec:` provider with shell script wrapper |
| Two-arm comparison | Native — all arms in one table |
| JSON handling | Must strip code fences in shell script |
| Grading | Assertion combinators (is-json, javascript, contains, llm-rubric) |
| Scaling | `file://` glob, `--filter-pattern`, `--ci`, eval history, web UI |

**Strength:** Multi-arm parallel comparison, web UI, eval history,
CI-native (`--ci` exit code).
**Weakness:** Must handle code fences manually; Node.js dependency;
YAML not interoperable with steward fixtures.

### Inspect AI

Not tested (requires API key, incompatible with Max subscription).

## Recommendation

**Use promptfoo for the Airflow skill-eval harness.**

Rationale:
1. Multi-arm comparison is a core requirement — promptfoo does this
   natively; steward runner requires multiple runs + manual comparison.
2. The 4-layer test matrix (skill effectiveness, AGENTS.md regression,
   coexistence, real-env discovery) maps directly to promptfoo's
   multi-provider architecture.
3. Scaling: `file://cases/*.yaml` + `--filter-pattern` + web UI +
   eval history covers the path from 3 cases to 100+.
4. CI integration: `npx promptfoo eval --ci` returns exit code 0/1.
5. Zero runner code to maintain — just YAML cases and shell scripts.

Trade-off: abandons steward fixture format interoperability.
This was agreed as a goal in the sync recap, but the practical
benefit (case sharing with steward) is low — Airflow's command-routing
cases are domain-specific and unlikely to transfer.

**Proposed next steps:**
1. Discuss with mentors: accept promptfoo as the eval framework,
   dropping steward fixture format requirement.
2. Expand to 10+ discriminating cases (provider tests, uv fallback
   scenarios, cross-package changes, breeze suite selection).
3. Investigate why `agents-proposed` still fails case-1 —
   analyze AGENTS.md holistically, not just line 26.
4. Set up `npx promptfoo eval --ci` in GitHub Actions for
   regression detection on AGENTS.md / SKILL.md changes.

## Appendix: Reproducing

```bash
# Full 7-arm matrix (21 calls, ~1 min)
cd dev/skill-evals-spike/promptfoo-test
npx promptfoo eval -c promptfooconfig.yaml --no-cache

# View results in browser
npx promptfoo view

# steward runner (3 cases, print mode)
python3 dev/skill-evals-spike/steward-test/runner.py \
    dev/skill-evals-spike/steward-test/evals/command-routing/

# steward runner (3 cases, automated)
python3 dev/skill-evals-spike/steward-test/runner.py \
    --cli "claude -p" \
    dev/skill-evals-spike/steward-test/evals/command-routing/
```
