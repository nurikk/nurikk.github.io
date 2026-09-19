---
title: Judge runs, don't read them
date: 2026-09-17 09:00:00 +0100
categories: [Agents]
tags: [evaluation, agent loops, reliability]
description: A small binary judge, a versioned rubric, and explicit terminal states make retrieved past runs useful without turning them into authority.
image:
  path: /assets/img/posts/judge-runs-dont-read-them.webp
  alt: Several execution traces converge on a precise decision gate.
---

When an agent can retrieve past runs, the first temptation is to show it a pile of transcripts and ask whether the current run looks complete. That produces plausible commentary, but it leaves the stopping decision inside an untestable paragraph.

A better design treats past runs as evidence and the judge as a narrow function. The runner owns execution. The judge answers one question: given the current evidence and the current rubric, should the run continue or stop?

## Retrieve evidence, not precedent

Past runs are useful for finding similar failures, successful tool sequences, and cases where a superficially complete response was missing a required step. Retrieve them by task shape, failure mode, and versioned environment. Keep their provenance: run ID, agent version, tool contract version, rubric version, and relevant outputs.

Do not let a retrieved transcript become an instruction. It is an example from an earlier execution. It may reflect a different policy, a different tool schema, or an accidental success. The current run still needs current validation.

The judge input can be compact:

```text
current task
required outcomes
observed user-visible result
required tool events
forbidden events
retrieved examples, with versions
```

There is no need to replay every old token. A structured summary is easier to inspect and less likely to smuggle stale instructions into the active context.

## Version the rubric

A judge is only as stable as its rubric. Store the rubric with a version and make each requirement observable. “The agent handled the request well” is not a check. “The confirmation page shows the submitted reference and no duplicate submission occurred” is closer to one.

A useful rubric distinguishes requirements from preferences. Requirements are binary conditions that block completion. Preferences can be scored separately or ignored by the terminal decision. Record why a requirement was judged true or false, and keep the evidence pointer that supports it.

Rubric changes are evaluation changes. When a new requirement is added, compare scores under both versions before changing production behavior. Otherwise a drop in completion may be a product regression, a stricter judge, or both.

## Make the terminal state explicit

The output should be deliberately boring:

```json
{
  "decision": "continue",
  "failed_checks": ["confirmation_visible"],
  "evidence": ["run.current.page"],
  "rubric_version": "2026-09-17"
}
```

The other valid decision is `done`. Do not use free-form labels such as `probably_done`, and do not infer completion from an empty explanation. The runner should reject unknown decisions, verify that every required check has a result, and apply a bounded retry or escalation policy when the judge cannot inspect the evidence.

A binary judge does not mean the underlying assessment is simple. Each check can be implemented by a deterministic assertion, a specialized grader, or a model-assisted comparison. The top-level contract stays binary so the runner has a clear transition. A failed check returns the run to an allowed next action. It does not invite an open-ended chain of retries.

Anthropic's [guide to evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) describes graders as logic that scores aspects of an agent's performance, including multi-turn runs with tools and an environment. That framing is useful here: separate the checks from the policy that decides how their results affect execution.

## Calibrate before trusting it

A judge can be consistently wrong. Build a calibration set with positive and negative examples, including near misses, partial tool success, stale evidence, and runs that stopped after an error. Have people label the set against the rubric, then compare judge decisions and disagreements.

Track false `done` decisions more aggressively than false `continue` decisions when stopping early can create a side effect or an incomplete user task. Review disagreements by check, task family, model version, and rubric version. A single aggregate accuracy number hides the failures that matter.

Use a second judge only when it answers a distinct question or provides a useful escalation path. Two models agreeing on the same ambiguous evidence is not independent validation. Deterministic checks should remain deterministic. A screenshot, tool log, or database record is stronger evidence than a judge's claim that the run probably did something.

## Keep retrieval out of the authority path

Past runs can suggest what to inspect next. They should not grant permission, satisfy a current policy requirement, or prove that a side effect committed. The current run needs current evidence. If the task involves a mutation, the server should verify the post-commit state directly and publish a durable event from that state. A judge can mark the workflow complete after that verification, but it cannot substitute for it.

The practical pattern is small: retrieve comparable runs, apply a pinned rubric, return `continue` or `done`, and make the runner enforce the result. The value comes from making judgment inspectable and repeatable, not from having the judge write a longer explanation.
