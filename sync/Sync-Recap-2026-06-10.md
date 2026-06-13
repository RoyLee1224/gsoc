### **Decisions**

- Eval harness (Task 4) comes before finalizing skill content (Task 2) — **agreed**
- Build a repeatable harness; reuse steward’s fixture format (step-config.json / report.md / expected.json), runner maintained separately in the Airflow repo — **agreed**
- Context-degradation experiments **deferred**.
  - Harness v1 scope: single-model, single-turn, with-skill vs without-skill comparison on basic mock data, using `claude -p --bare` to isolate the system prompt and control what context each arm sees. pass@k / pass^k, multi-model, and multi-turn are explicitly v2+.
- AGENTS.md contradiction (always-breeze vs uv-first): **resolution waits for eval evidence** — the contradiction itself is the demo case for why the eval is needed

### **Next Steps**

- Tracking issue + dev-list post for the eval harness — v1 scope, open questions.
- Steward skill-evals deep-dive → design note
- Harness v1 scaffold — steward-compatible fixtures, separate runner in the Airflow repo (proposing dev/skill-evals/ in the PR).

### **Open question (for @Jarek Potiuk)**

Community agent-failure-case collection — discussed with Jason earlier, raising it here for your thoughts.

The original idea was lightweight reporting channels — a PR-template section for agent mistakes contributors hit while developing (wrong command, fixed during the session), an issue template for standalone failures (agent misjudges which command to use and loops), plus a `kind:agent-case-study` label to make all of it findable.

Well-diagnosed failures convert directly into eval scenarios — real failures tell us which rules agents actually break, rather than which ones we guess.

**I’ve since cooled on the template part**: it adds workload for maintainers to triage, at a time when maintainers are trying so hard to reduce the queue. The label alone might be the right start — zero new process, and it still lets us spot patterns.

**My focus stays on the eval harness**, so no action needed now — but longer-term this could be a good pipeline for real eval cases. Curious what you think.
