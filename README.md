# Apache Airflow Contribution & Verification Agent Skills

## Google Summer of Code 2026 Final Report

| | |
| --- | --- |
| Contributor | [Jhe Chen Li](https://github.com/RoyLee1224) |
| Mentor | [Jason Liu](https://github.com/jason810496) |
| Organization | [Apache Software Foundation](https://www.apache.org/) |
| Project | [Apache Airflow](https://github.com/apache/airflow) |
| Program | [Google Summer of Code 2026](https://summerofcode.withgoogle.com/) |
| Project period | May 25–August 24, 2026 |
| Community discussion | [Airflow development mailing-list thread](https://lists.apache.org/thread/vggoo6yhx2x7nj6w0khc0ynwt9xrbrqq) |

## Summary

This project explored how Apache Airflow can provide reliable contribution and
verification guidance to coding agents.

The original proposal focused on contributor-facing skills: helping an agent
select the correct development environment, follow Airflow's contribution
workflow, and verify its changes. Early experiments revealed a more fundamental
problem. Adding instructions was easy, but there was no reliable way to tell
whether those instructions were correct, remained synchronized with Airflow, or
actually improved agent behavior.

The project therefore established three complementary guidance-quality layers:

1. **Sync** documentation and agent instructions from a shared source of truth.
2. **Validate** factual claims, such as documented commands, against the code
   that defines them.
3. **Measure** behavioral effects with reproducible, Airflow-specific agent
   evaluations.

Four contributions implementing these layers were merged into Apache Airflow.
The resulting infrastructure provides a foundation for introducing smaller,
on-demand contribution skills without relying only on intuition.

The initial results and proposed next steps were shared with the Airflow
community in this [public development mailing-list
thread](https://lists.apache.org/thread/vggoo6yhx2x7nj6w0khc0ynwt9xrbrqq).
Community feedback will help prioritize the follow-up work described below.

## Scope evolution

The initial proof-of-concept skill covered environment detection, test-runner
selection, static checks, and parts of the contribution workflow. In broader
one-shot experiments, however, the difference between the with-skill and
without-skill arms fell to approximately `+0.07`; the baseline model already
passed many of the clean-context cases.

This made it difficult to justify adding more instructions without a repeatable
measurement method. After discussion with my mentors, I prioritized the eval
harness before finalizing the skill content. The skills remain part of the
project direction, but they are now treated as hypotheses that should be
validated through before-and-after evaluations.

This was an evidence-driven change in implementation order: build the mechanism
for verifying guidance first, then use it to decide what guidance should be
always loaded, path-specific, or available on demand.

## Completed contributions

| Layer | Contribution | Status | Outcome |
| --- | --- | --- | --- |
| Sync | [#68204: Sync `AGENTS.md` commands from contributing docs via prek hook](https://github.com/apache/airflow/pull/68204) | Merged | Made the contributing documentation the source of truth for the generated command block in `AGENTS.md` |
| Validate | [#70186: Validate documented commands against the Breeze and prek registries](https://github.com/apache/airflow/pull/70186) | Merged | Added a deterministic check for invalid Breeze command paths, flags, and prek hook IDs |
| Measure | [#69308: Add eval harness for testing `AGENTS.md` changes](https://github.com/apache/airflow/pull/69308) | Merged | Added isolated before/after agent evaluations using real Airflow scenarios |
| Measure | [#70120: Evaluate `AGENTS.md` and skills with Codex](https://github.com/apache/airflow/pull/70120) | Merged | Added Codex as a second supported eval runtime |

### 1. Sync: one source of truth for commands

Before [#68204](https://github.com/apache/airflow/pull/68204), the commands in
the contributing documentation and root `AGENTS.md` were maintained
independently. A correct instruction could be updated in one place and remain
stale in the other.

The PR added marked source sections to the contributing documentation and a
prek hook that regenerates the corresponding `AGENTS.md` command block. CI now
detects drift automatically. This makes later modularization safer because
skills and agent instructions can be derived from an explicit source rather
than copied manually.

### 2. Validate: keep documentation aligned with reality

Synchronization alone does not guarantee correctness: an incorrect source
command can be propagated consistently. One real example was the documented
`breeze selective-checks` command, which never existed; the actual command is
`breeze ci selective-check`.

[#70186](https://github.com/apache/airflow/pull/70186) added a prek check that
extracts documented Breeze and prek commands and validates them against the
actual command registries. It checks command paths, Breeze options, and prek
hook IDs, while accounting for placeholders and passthrough commands.

Running the checker over the existing documentation found ten stale or mistyped
references representing seven distinct invalid commands. Some had remained in
the documentation for between six and twenty months. The PR corrected all of
them and added the check to prevent the same class of drift from returning.

### 3. Measure: an Airflow-specific agent eval harness

[#69308](https://github.com/apache/airflow/pull/69308) added
`dev/skill-evals/`, a harness for asking whether a proposed change to
`AGENTS.md` or a skill actually changes agent behavior.

For each experiment, the harness creates isolated git worktrees, gives every
arm the same repository and task, requests structured output, and compares the
result with executable assertions. It supports repeated runs, a no-guidance
baseline, working-tree comparisons, result export, and a freshness gate that
requires guidance changes to be evaluated.

The first runtime used the Claude Agent SDK. [#70120](https://github.com/apache/airflow/pull/70120)
extended the same cases and arms to Codex, using a fresh thread, read-only
sandbox, disabled network access, and schema-constrained output for every case.
This begins separating Airflow-specific guidance effects from the behavior of a
single agent runtime.

## Evaluation showcases

### Newsfragment decisions

Airflow maintainers observed that agents and contributors repeatedly added
newsfragments for changes that were not clearly user-facing, creating review
cleanup. [#67982](https://github.com/apache/airflow/pull/67982) introduced a
golden rule: when uncertain, omit the newsfragment and let a maintainer request
one during review.

The eval harness converted this feedback into scenarios drawn from real PRs.
Four ambiguous cases where a newsfragment should be omitted were repeated three
times per arm:

| Scenario | With `AGENTS.md` | Without `AGENTS.md` |
| --- | ---: | ---: |
| Provider monitoring-pod leak | 3/3 | 0/3 |
| API query optimization | 3/3 | 1/3 |
| Scheduler fix | 2/3 | 0/3 |
| i18n cache fix | 1/3 | 0/3 |
| **Total** | **9/12** | **1/12** |

The guidance materially improved the decisions, but ambiguous fixes remained
unreliable. Two positive controls, where a feature newsfragment should be
created, were also added; both arms passed those cases 6/6. This prevents an
"always omit" policy from passing the suite.

The result illustrates both the value and the limit of the guidance: it changes
behavior substantially, but it does not make every judgment deterministic.

### Measuring when MCP provides value

I also used the harness to examine the development MCP server proposed in
[#69381](https://github.com/apache/airflow/pull/69381). The experiment used a
frozen runtime reproduction of the real bug in
[#39801](https://github.com/apache/airflow/issues/39801). Each arm received the
same symptom-only debugging task and was scored against three expected
findings.

| Arm | Result | Turns | Tokens |
| --- | --- | ---: | ---: |
| MCP tools | Passed 3/3 assertions | 10 | 27.8k |
| CLI + documented workflow | Passed 3/3 assertions | 7 | 27.4k |
| Bare CLI | Passed 3/3 assertions | 28 | 54.7k |
| Source inspection only | Failed at the turn limit | — | — |

For this scenario, runtime access was decisive. The source-only arm found the
relevant scheduler code but could not identify the incorrect recorded task
state. MCP and a documented CLI/API workflow both reached the correct diagnosis
efficiently. Bare CLI access also succeeded, but discovery and authentication
friction roughly doubled its token use.

This is a sealed `n=1` experiment, not a general recommendation to choose MCP,
CLI documentation, or skills. It does not measure multi-call aggregation,
typed-interface advantages, delivery reliability, or long-session effects. Its
main contribution is demonstrating that the eval framework can separate the
value of runtime access from the mechanism used to provide it and report the
trade-off with concrete evidence.

## How to use the eval harness

The upstream documentation and implementation live in
[`dev/skill-evals/`](https://github.com/apache/airflow/tree/main/dev/skill-evals).

Run the default Claude evaluation:

```bash
prek run run-skill-eval --hook-stage manual --all-files
```

Repeat every case to reduce sensitivity to nondeterminism:

```bash
EVAL_REPEAT=3 prek run run-skill-eval --hook-stage manual --all-files
```

Run the same cases with Codex:

```bash
prek run run-skill-eval-codex --hook-stage manual --all-files
```

The harness exports a JSON report under `files/skill-evals/` and can display
the results through promptfoo's local viewer.

## Current state

The four primary contributions listed above are merged into Apache Airflow.
The following experiments and extensions were developed during the project but
are not yet upstream deliverables:

- **OpenCode runtime:** a working runtime adapter on
  [`feat/skill-eval-opencode-runtime`](https://github.com/RoyLee1224/airflow/tree/feat/skill-eval-opencode-runtime).
- **MCP usefulness eval:** the sealed scenario, harness extension, raw traces,
  and four-arm experiment on
  [`feat/mcp-usefulness-eval`](https://github.com/RoyLee1224/airflow/tree/feat/mcp-usefulness-eval).
- **Contribution skills:** proof-of-concept guidance for environment and test
  routing exists, but the final skill split has not been proposed upstream
  because it should first be evaluated against the current root guidance.

This distinction is intentional: merged infrastructure and experimental
evidence are reported separately from future design proposals.

## Future work

### Slim the root `AGENTS.md`

At the end of the project, Airflow's root `AGENTS.md` contained 522 lines and
35,925 bytes. Its `Commits and PRs` section alone contained 307 lines—about 59%
of the file's lines—and the root file exceeded Codex's default 32,768-byte
project-document limit.

The next experiment will compare at least three arms:

1. the current root `AGENTS.md`;
2. a slimmer root file;
3. the slimmer root file with relevant workflows available through on-demand
   skills.

The goal is not simply to minimize the file. It is to find the smallest
always-loaded context that preserves Airflow-specific correctness.

### Evaluate focused contribution skills

One candidate decomposition is:

- `airflow-code-change-verification`: select prek, uv, or Breeze checks and
  decide whether a newsfragment is required;
- `airflow-pr-draft-summary`: apply Airflow's commit and PR conventions and
  Gen-AI disclosure requirements;
- `airflow-publish-changes`: commit, push, or open a PR only after an explicit
  user request.

These boundaries are hypotheses. Before proposing them upstream, the eval
should verify that a slim-root-plus-skills arm preserves task success and
instruction compliance without adding discovery failures.

### Maintain hierarchical guidance

Airflow currently has fourteen tracked `AGENTS.md` files: one root file and
thirteen path-specific files. Future cases should independently test:

- **discovery:** whether the applicable nested guidance is loaded; and
- **behavior:** whether that guidance changes the relevant decision.

This would help determine whether each rule belongs in the root file, a nested
file, an on-demand skill, or nowhere because it is duplicated or obsolete.

### Broaden the evidence

Further work should add repeated MCP trials, multi-call cases, misleading
runtime-state cases, and cross-runtime replication. Larger community tasks can
be tracked in a meta issue, with smaller cases contributed independently.

## Challenges and lessons learned

- **Synchronizing documentation does not prove it is correct.** Generation and
  registry validation solve different failure modes.
- **Static facts and behavioral judgments need different tests.** Deterministic
  checks should validate command existence; agent evals should measure judgment
  and workflow behavior.
- **A single successful run is weak evidence.** Repetition, sealed arms, explicit
  assertions, and honest limitations are necessary for interpretable results.
- **Runtime access and its delivery mechanism are separate variables.** The MCP
  experiment showed why an eval should isolate them rather than treating tool
  availability as a binary property.
- **More context is not automatically better.** Large always-loaded instruction
  files introduce token cost, truncation risk, conflicting rules, and ongoing
  maintenance work.

## Acknowledgements

I am grateful to my mentor, Jason Liu, for regular technical guidance, reviews,
and help refining the project scope. I also thank the Apache Airflow maintainers
and contributors who reviewed the pull requests, provided real failure cases,
and challenged the experimental assumptions. Their feedback turned the initial
skill prototypes into infrastructure that can support future work across the
community.

I plan to continue contributing to Apache Airflow after GSoC, beginning with the
root-guidance experiment and follow-up evaluation cases described above.
