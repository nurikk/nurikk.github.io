---
title: Make the harness earn its place
date: 2026-09-20 18:00:00 +0100
categories: [Agents]
tags: [agent harnesses, evaluation, coding agents]
description: Planning, memory, and specialized tools belong in an agent harness only when controlled tests show which failure they prevent.
image:
  path: /assets/img/posts/make-the-harness-earn-its-place.webp
  alt: Modular agent components pass through a measurement gate before joining an execution loop.
---

Agent harnesses tend to grow by anecdote. The agent forgets a step, so someone adds a plan. It reads too much, so someone adds memory. It mistypes a shell command, so the harness gets another specialized tool. Each addition sounds sensible. Six months later, nobody can say which parts improve completed work and which parts merely add tokens, latency, and new failure modes.

That is not an argument for a tiny harness. It is an argument for treating every harness feature as a testable intervention.

## Start with the failure, not the feature

A harness component needs a job stated in observable terms. "Improve reasoning" is not a job. "Prevent the agent from stopping before its first edit" is. So is "keep runs inside a 32K context window" or "stop unsupported package-manager commands before execution."

The difference matters because a component can change behavior without improving outcomes. A planning step may produce a cleaner transcript while lowering the solve rate. A recall tool may preserve old observations that the model never asks for. Five narrow file tools may turn one shell operation into several model round trips.

Agent Zero recently reported six changes to prompts, memory, runtime guidance, and time awareness. None earned promotion into its general harness. The team kept the model, tasks, starting state, and configuration fixed, then judged changes by completed behavior rather than how reasonable they sounded.[1] That is a healthier default than merging a prompt tweak after one convincing run.

Write the feature hypothesis before implementation:

```text
Failure: the agent exits before changing code.
Intervention: inject and persist a short task plan.
Pass condition: fewer premature exits on the fixed task set,
without reducing solved tasks or raising cost per solved task beyond the budget.
```

The pass condition forces an honest trade. Planning can help and still be too expensive. A cheaper run can be worse if it stops before doing the work.

## The useful harness depends on the model

A recent study varied planning, action space, and context management across four models and two coding benchmarks, covering 176 matched settings. Its result was conditional, not a universal architecture: planning helped weaker models continue far enough to attempt edits, while stronger models mostly used it to remove redundant work; predefined tools helped models with weaker shell skills, while bash-capable models could perform well with a simpler interface at lower cost.[2]

This makes "best harness" the wrong unit of comparison. The deployed unit is a model, a harness, a context budget, and a task distribution. Change the model and an old scaffold may become overhead. Shrink the context window and compaction may move from optional to necessary.

The same study found that context management mattered most under tight budgets because it prevented overflow failures. Cheap removal of stale tool output before model-generated summarization gave the best overall efficiency in the tested setup. Making removed observations retrievable added machinery, but models rarely recalled them and accuracy did not improve.[2]

That finding should not become a new rule that recall is useless. It should change the burden of proof. If a production agent needs recall, collect cases where missing an older observation caused a wrong action. Then test whether the retrieval path recovers those cases, whether the model actually calls it, and whether unrelated retrieval makes other tasks worse.

## Measure outcomes and behavior separately

An end-to-end score answers the question that matters most: did the agent finish the task? It is usually poor at telling you why a change helped or failed. Reading entire transcripts can explain a few runs, but it is slow and easy to rationalize after the fact.

Use two layers. The macro suite contains representative tasks with real acceptance checks. The behavioral suite contains narrow regressions for known harness failures: inspected the authoritative file before editing, used the repository's package manager, ran the relevant validator, stayed inside the allowed paths, and did not claim success after a failed command.

Google's harness-engineering guidance recommends this split. Its behavioral evaluations assert discrete intermediate actions and run quickly, while larger end-to-end evaluations remain the check on final performance.[3] The small checks shorten diagnosis time; they do not replace outcome validation.

That distinction prevents a common mistake. Suppose a new instruction makes the agent run tests more often. The behavioral check turns green, but the task solve rate falls because the agent spends its budget on irrelevant suites. The instruction fixed the measured behavior and hurt the product. Keep the behavior check, but do not promote the intervention.

Avoid overfitting the path as well. For a deterministic requirement such as "never create `package-lock.json` in this Bun repository," a strict assertion is appropriate. For a repair that can be reached through several valid investigations, checking one exact tool sequence will punish harmless variation. Assert the invariant or outcome, not your favorite transcript.

## Tests can agree with the same mistake

Even a task-specific test is not automatically independent evidence. If one trajectory misreads the issue, writes the patch, writes a test for that patch, and decides when to stop, the patch and test can share one blind spot.

ExecCritic separates those roles. A test agent works from the issue and buggy checkout, the harness requires a clean failure and freezes the accepted test, and a repair agent may change source code but not the test. In its SWE-bench Verified experiments, weak generated tests reduced the fixed repair agent's resolved rate from 61.2% to 57.3%, while stronger generated tests raised it to 65.3%.[4] Execution feedback helped only when the test represented the requested behavior.

The production lesson is narrower than "use two agents." Freeze acceptance evidence before the implementation can edit it. Make a regression test fail on the known-bad state. Restrict the repair step from weakening that test. Keep a broader evaluator or human review as final authority because a clean failure on the old code does not prove that the assertion is the right one.

This principle also applies to harness changes. The person or agent proposing a memory layer should not be allowed to replace the evaluation set with examples tailored to that layer. Preserve held-out tasks and existing successes. Otherwise the test suite becomes a sales demo for the intervention.

## Run ablations like ordinary engineering experiments

A useful harness experiment is deliberately boring:

1. Save the task set, starting repository states, model configuration, tool schemas, budget, and acceptance checks.
2. Record the baseline solve rate, failure categories, cost per solved task, and run-time distribution.
3. Change one component.
4. Repeat enough runs to expose nondeterminism.
5. Inspect regressions and gains by failure category, not only in aggregate.
6. Promote the change only if it clears the prewritten threshold.

Keep the old path easy to restore. Version prompts and schemas. Record which harness revision produced each run. A model upgrade deserves the same comparison because it can change the value of tools and scaffolds even when the harness code is untouched.

Do not optimize token count in isolation. Fewer tokens can mean better context hygiene, or it can mean that the agent gave up early. Do not optimize tool-call count in isolation either. One dense shell command may be efficient and hard to govern; several typed calls may cost more and provide the authorization boundary the product needs. The metric has to match the failure and the operational constraint.

A harness should be the smallest system that reliably handles the failures you have evidence for. Keep components that prevent a measured failure at an acceptable cost. Remove components that survive only because their names sound like capabilities. When the model, budget, or task mix changes, make them earn their place again.

## Sources

[1] https://x.com/Agent0ai/status/2099876602121212330 — Open the Harness. Learn What Makes an Agent Work.
[2] https://arxiv.org/html/2609.20804 — An Empirical Study of Harness Design for Coding Agents
[3] https://developers.googleblog.com/the-anatomy-of-harness-engineering-how-to-evaluate-iterate-and-guard-ai-coding-agents — The Anatomy of Harness Engineering
[4] https://arxiv.org/html/2609.09133v1 — ExecCritic: Learn to Test, Test to Improve for Coding Agents
