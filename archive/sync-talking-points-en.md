# Sync Talking Points (English)

> 按 sync prep 的順序，每段都有可以直接講的英文。
> 不用背 — 掃一遍知道關鍵字怎麼說就好。

---

## 開場

> Let me give you a quick status update on the GSoC deliverables,
> and then I'd like to discuss the eval direction.
>
> The prek sync hook PR is up — Task 3.
> For Task 4, I ran three rounds of prototype evals and found
> something worth discussing.
> Tasks 1 and 2 have a basic implementation in the skill file,
> but I want to check the direction before going deeper.

---

## Part 1: Eval 發現

### 講 AGENTS.md 矛盾

> I found a contradiction in AGENTS.md.
> Line 26 says "never run pytest on the host, always use breeze."
> But lines 35 onward list a bunch of `uv run` commands — which
> run pytest directly on the host.
>
> In my evals, I actually observed the agent citing line 26
> to justify choosing breeze when uv was the correct runner.

### 講 eval 數據

> I ran three iterations of with-skill versus without-skill evals.
>
> First iteration, three cases — the skill showed a clear advantage.
> Pass rate was 1.0 with the skill versus 0.6 without.
>
> But when I expanded to more cases in iterations two and three,
> the delta dropped to just plus 0.07. The baseline without the
> skill was already 0.80 to 0.93.
>
> Most cases were non-discriminating — the model got them right
> either way.

### 講關鍵觀察

> The key takeaway: on one-shot tasks, the model already performs
> well. The skill adds very little in a clean context.

---

## Part 2: 假說

### 講假說

> So my hypothesis is: the problem isn't skill content quality.
> The problem is context degradation over long sessions.
>
> There's literature supporting this — SlopCodeBench found that
> quality guidance improves the starting point but doesn't change
> the degradation rate. Lost in the Middle shows a U-shaped curve
> for information utilization in long contexts.
>
> But I don't have Airflow-specific data yet. My evals were all
> one-shot.

### 問 Jarek

> Have you observed something similar in steward? Like, agents
> forgetting skill rules after a long session?
>
> Do you think this hypothesis is worth pursuing, or should I
> focus on finishing the other deliverables first?

### 講 eval 框架（解釋 steward eval 給 Jason）

> (To Jason) Quick context — the steward repo has a skill-evals
> framework. It's pretty simple: extract a section from SKILL.md,
> pair it with mock input, pipe it to `claude -p`, get back JSON,
> and diff against expected output. Justin just merged field-aware
> grading two weeks ago — prose fields like "rationale" get judged
> by Haiku for semantic equivalence instead of exact string match.

### 講自己的方案

> I want to use the same fixture format — step-config, report.md,
> expected.json — so cases are interoperable. But I'd like to
> maintain a separate runner in the Airflow repo.
>
> Three reasons:
>
> First, I need to test how AGENTS.md interferes with the skill.
> I can use `claude -p --bare` with `--append-system-prompt-file`
> to control exactly what context the model sees.
>
> Second, I'll eventually need multi-turn experiments to validate
> the context degradation hypothesis. The steward runner is
> single-turn only.
>
> Third, Airflow's runner selection logic is more complex than
> triage classification — there are three paths: uv, breeze,
> and prek.
>
> Does that make sense? And where should the runner live —
> `dev/skill-evals/`?

---

## Part 3: 短期 action items

> Regardless of the hypothesis, there are things I can do right now.
>
> One — fix the line 26 contradiction in AGENTS.md.
> The eval already proved it causes wrong decisions.
> Should I include that in the current PR or open a separate one?
>
> Two — address the PR review feedback. I need to add
> `additional_dependencies` for the prek hook and align
> the script style with other prek scripts.
>
> Three — for Task 1, the environment detection in the skill
> uses `test -f /.dockerenv`. Is that sufficient, or do you
> want a more robust mechanism like a helper script?

---

## Part 4: Magpie 邊界

> One more thing — Magpie's roadmap includes Pairing and Mentoring,
> which are expanding toward the contributor side. My work is also
> contributor-side.
>
> When is Pairing / Mentoring starting? What's the scope?
> Would my eval framework conflict with that?
>
> I'm using the `airflow-*` prefix for my skills, not going
> through the Magpie snapshot. Is that okay?

---

## 常用句型（接話 / 追問 / 確認）

| 情境 | 英文 |
|------|------|
| 確認理解 | "Just to make sure I understand — you're saying...?" |
| 追問細節 | "Could you elaborate on that?" |
| 表達同意 | "That makes sense. I'll go with that." |
| 表達不確定 | "I'm not sure about this part — what do you think?" |
| 接受建議改方向 | "Good point. I'll adjust my approach." |
| 確認 action item | "So the action item is... — did I get that right?" |
| 時間不夠時 | "I have a couple more items — should I send them async?" |
| 結尾 | "Thanks for the feedback. I'll follow up on [X] by [date]." |
