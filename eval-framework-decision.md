<!--
 Licensed to the Apache Software Foundation (ASF) under one
 or more contributor license agreements.  See the NOTICE file
 distributed with this work for additional information
 regarding copyright ownership.  The ASF licenses this file
 to you under the Apache License, Version 2.0 (the
 "License"); you may not use this file except in compliance
 with the License.  You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

 Unless required by applicable law or agreed to in writing,
 software distributed under the License is distributed on an
 "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 KIND, either express or implied.  See the License for the
 specific language governing permissions and limitations
 under the License.
-->

# Eval Framework for Agent Guidance

## Problem

When a contributor modifies `AGENTS.md` or a skill file (`SKILL.md`),
there is no systematic way to verify the change improves agent
behavior or avoids regressions. The contributor must manually test
with Claude, eyeball responses, and hope nothing broke.

## Proposal

Use [promptfoo](https://github.com/promptfoo/promptfoo) with the
Claude Agent SDK provider as a dev-time eval tool. The harness
compares agent behavior **before** and **after** a guidance change,
using the same set of probe cases.

### What it does

```bash
# After modifying AGENTS.md or SKILL.md:
./dev/skill-evals/eval.sh

# Output:
# Changes detected (vs main):
#   AGENTS.md — modified
#   SKILL.md  — unchanged
#
#          | before | after
# case-1   | PASS   | FAIL   ← regression
# case-2   | PASS   | PASS   ← no impact
# case-3   | FAIL   | PASS   ← improvement
```

### How it works

1. `eval.sh` extracts the `main` branch versions of `AGENTS.md` and
   `SKILL.md` via `git show`, and copies the working tree versions.
2. It builds temporary working directories — one per arm — with the
   appropriate files placed where the Agent SDK discovers them.
3. promptfoo runs each case against both arms in parallel and
   produces a side-by-side comparison.
4. Temporary directories are cleaned up on exit.

### Architecture

```
dev/skill-evals/
  eval.sh                       # entry point
  promptfooconfig.yaml           # quick mode: 2 arms (before vs after)
  promptfooconfig.full.yaml      # full mode: 5 arms (+ isolation arms)
  cases/
    command-routing.yaml         # cases grouped by concern
```

**Quick mode (default):** 2 arms — before (main) vs after (working tree).
Answers "did my change cause a regression?"

**Full mode (`--full`):** 5 arms — adds baseline (no guidance),
skill-only (skill without AGENTS.md), and agents-only (AGENTS.md
without skill). For periodic analysis of how guidance components
interact.

### Adding cases

Add entries to an existing file in `cases/` or create a new file.
Cases are YAML — no directory structure, no configuration changes:

```yaml
# cases/command-routing.yaml
- description: "Provider test: uv fails, fallback to breeze"
  vars:
    request: |
      Run amazon provider tests.
      uv failed with: error: libpq-dev not found
  assert:
    - type: javascript
      value: 'output.runner === "breeze"'
```

All arms run every case automatically.

### Prerequisites

- `ANTHROPIC_API_KEY` environment variable
- `npx` (Node.js, already present for UI development)

## Triggering eval on guidance changes

Eval requires LLM API calls — too expensive and slow for a
pre-commit hook. Instead, the harness integrates at two levels:

### Prek hook: reminder (no API cost)

A prek hook detects when `AGENTS.md` or `SKILL.md` is modified
and prints a reminder. No API calls, no latency — just a message:

```
⚠  AGENTS.md modified — run ./dev/skill-evals/eval.sh before pushing
```

This fits the existing prek pattern. The repo already has a
`generate-agent-skills` hook that triggers on `AGENTS.md` changes
(`.pre-commit-config.yaml` line 936). The reminder hook uses the
same file pattern.

### PR review: eval results in description

Contributors who modify guidance files include eval results in the
PR description — before/after pass rates, which cases regressed,
which improved. Reviewers can see the impact at a glance.

```markdown
## Eval results

| Case | before | after |
|------|--------|-------|
| Core unit test (uv) | PASS | PASS |
| Helm test (breeze) | PASS | PASS |
| mypy (prek) | FAIL | PASS |

Pass rate: 2/3 → 3/3
```

This keeps eval voluntary (contributors run it, not CI), while
making the results visible during review.

## Spike results

3 cases, 7 arms (initial spike with manual arm configuration):

| Arm | Case 1 (contradiction) | Case 2 (Helm) | Case 3 (mypy) |
|-----|:---:|:---:|:---:|
| baseline | FAIL | PASS | FAIL |
| skill-only | PASS | PASS | PASS |
| agents-current | FAIL | PASS | PASS |
| agents-proposed | FAIL | PASS | PASS |
| skill+agents-current | PASS | PASS | PASS |
| skill+agents-proposed | PASS | PASS | PASS |
| real-env | PASS | PASS | PASS |

Key findings:

1. **Skill is effective.** Baseline 1/3 → skill-only 3/3.
2. **Fixing line 26 alone is not enough.** `agents-proposed` still
   fails case-1. The 470-line AGENTS.md likely creates a
   breeze-preference bias beyond one line.
3. **Skill overrides the contradiction.** When skill and AGENTS.md
   coexist, the skill's routing rules take priority.
4. **Real-env confirms skill is loaded.** Matches skill+agents
   results.

The cases are not yet discriminating enough — 5 of 7 arms scored
3/3. Expanding to 10+ cases is the immediate priority.

## Context degradation

promptfoo handles single-turn evaluation well. For multi-turn
context degradation (does the skill hold up after 15 turns of
real work?), a rough proxy is possible by varying prompt length:

```yaml
- vars: { filler: "{{file://fillers/40k.txt}}", probe: "..." }
```

True session-accumulation testing requires a custom Agent SDK
harness — deferred to a later phase, contingent on the rough proxy
showing signal.

## Vendor independence

promptfoo is owned by OpenAI (acquired 2026-03). The harness avoids
architectural dependency:

- Investment is in test cases (domain knowledge in YAML), not runner
  infrastructure.
- No CI integration — runs manually, on demand.
- Cases are portable — a 50-line Python script can replicate the
  eval loop. Migration cost: ~1 day.
- MIT license permits forking.

## Open questions

- Does the `anthropic:claude-agent-sdk` provider require API key
  access for GSoC contributors, or is there a project key?
- Is a prek reminder hook for guidance changes worth adding now,
  or after the case suite reaches a useful size?
