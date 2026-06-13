# GSoC Sync — 2026-06-10

## Links

- [AIP-to-user-stories skill PR #65776](https://github.com/apache/airflow/pull/65776)
- [Sync AGENTS.md commands via prek hook PR #68204](https://github.com/apache/airflow/pull/68204)
- [steward skill-evals field-aware grading PR #370](https://github.com/apache/airflow-steward/pull/370)

---

## Progress by Core Task

### Task 3: Syncing with Source of Truth (via prek) — PR ready

PR #68204: prek hook syncs AGENTS.md Commands section from
contributing-docs RST markers. Review feedback pending
(missing `additional_dependencies`).

**No blockers. Just need review.**

### Task 1: Environment Awareness & Detection — PoC done, blocked by AGENTS.md contradiction

SKILL.md has a basic detection check (`test -f /.dockerenv`)
and a workflow decision tree for routing commands (uv / breeze / prek).
This is a proof of concept, not a finished mechanism.

The original issue framed this as "host vs container" detection.
But in practice, agents always run on the host — the real question
is which command to use for a given task.

**The problem: AGENTS.md has three conflicting instructions:**
- Line 26: "Never run pytest on the host — always use breeze"
- Line 35-37: `uv run --project <PROJECT> pytest ...`
- Line 38: "If uv tests fail... run with breeze" (uv first, breeze fallback)

I can't design the routing logic until I know which rule is correct.
My eval observed agents citing line 26 to justify choosing breeze
when uv was the right answer.

**Question: Which rule should we follow? What is the intended
mental model — "uv first, breeze fallback" or "always breeze"?
Once that's settled, I can fix line 26 and build the routing
logic on solid ground.**

### Task 2: Workflow Skills — PoC done, blocked on eval

I have a local PoC skill (airflow-contribution) covering:
- Static checks (prek commands)
- Test runner selection (uv / breeze routing)
- Breeze suite commands

Not yet done: system verification (stretch goal).

I ran 3 rounds of prototype evals against this skill.
The result: **with-skill vs without-skill delta was only +0.07.**
The model already gets most cases right without the skill.

This aligns with SlopCodeBench's finding — quality guidance
improves the starting point but doesn't change the degradation
rate. Having a skill doesn't guarantee better performance.

**So I believe eval should come before the skill, not after.**
Without a repeatable eval harness, I can't tell whether skill
changes actually improve agent behavior. I'd be writing skill
content blind.

**Question: Do you agree that building the eval harness (Task 4)
should be prioritized over finalizing the skill content (Task 2)?**

### Task 4: Evaluation & Test Harness — prototype done, direction question

#### What I did

3 rounds of prototype evals using skill-creator, 13 cases total.
With-skill vs without-skill comparison.

| Iteration | Result |
|-----------|--------|
| iter 1 (3 cases) | with-skill 1.0 vs without 0.6 |
| iter 2 (5 cases) | delta dropped to +0.07 |
| iter 3 (5 cases) | delta +0.07, baseline already 0.80-0.93 |

**Key finding: on one-shot tasks, the model already performs well.
The skill adds very little in clean context.**

**Question: Should I build a repeatable eval harness — similar to
steward's skill-evals — so we can reproduce and track these
results systematically, rather than relying on one-off prototype runs?**

#### My hypothesis

The real problem is context degradation over long sessions,
not skill content quality. Literature supports this
(SlopCodeBench, Lost in the Middle), but I don't have
Airflow-specific data yet.

**Question: Have you seen similar degradation in steward?
Is this worth pursuing?**

#### Eval framework direction — the key question

I looked at steward's skill-evals framework
(for Jason: it extracts a SKILL.md section as prompt,
pairs with mock input, pipes to `claude -p`, diffs JSON
output against expected.json. Justin's PR #370 added
Haiku-based grading for prose fields).

**My proposal:**

Use the same **fixture format** (step-config.json, report.md,
expected.json) — so cases are interoperable.

Maintain a **separate runner** in the Airflow repo.

Reasons for a separate runner:
1. Need to test AGENTS.md + skill coexistence
   (`claude -p --bare` + `--append-system-prompt-file`
   to control isolation levels)
2. Need multi-turn experiments for context degradation
   (steward runner is single-turn)
3. Airflow's routing logic is more complex
   (uv / breeze / prek — three paths)

**Questions:**
- Fixture format compatible with steward, runner maintained
  separately in Airflow — does this make sense?
- Where should it live? `dev/skill-evals/`?

### Task 5: Documentation — not started

AGENTS.md slimming plan exists but not yet implemented.
Will start after Task 3 PR lands and line 26 is resolved.

---

## Magpie boundary

Magpie's Pairing / Mentoring is expanding toward contributor side.
My work is also contributor-side.

- When is Pairing / Mentoring starting? What's the scope?
- My skills use `airflow-*` prefix, not Magpie snapshot. OK?

---

## Summary of questions

- AGENTS.md contradiction: which rule is correct? "always breeze" or "uv first, breeze fallback"?
- Once resolved, fix line 26 in current PR or separate?
- Agree that eval harness (Task 4) should come before finalizing skill (Task 2)?
- Should I build a repeatable harness like steward's to verify these findings?
- Context degradation hypothesis worth pursuing?
- Fixture format compatible with steward, runner maintained separately in Airflow — does this make sense?
- Runner location: `dev/skill-evals/`?
- Magpie Pairing/Mentoring timeline and overlap?

---

## Takeaways & Action Items

- (fill during meeting)
