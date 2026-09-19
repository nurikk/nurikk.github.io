---
title: Load tools when you need them
date: 2026-09-15 09:00:00 +0100
categories: [Agents]
tags: [tools, progressive disclosure, context engineering]
description: Progressive disclosure keeps a large tool catalog out of the working context while preserving a clear path to the capability an agent needs.
image:
  path: /assets/img/posts/load-tools-when-you-need-them.webp
  alt: A compact index opens selected modules from a folded catalog.
---

A tool definition is input, even when the model never calls the tool. Names, descriptions, parameter schemas, examples, and policy notes all compete for context. Sending every tool on every turn makes the agent start with a catalog instead of a task.

The fix is not to hide tools randomly. It is to make discovery cheap, activation explicit, and authorization independent from model choice.

## Show an index before the manual

Progressive disclosure gives the model a small capability index first. Each entry needs a useful name and description, enough to answer one question: could this capability help with the current task?

The [Agent Skills implementation guide](https://agentskills.io/client-implementation/adding-skills-support) describes three levels. The catalog is available at session start. Full instructions arrive when a skill is activated. Supporting references, scripts, and assets are loaded only when an instruction points to them. Its rough guidance is about 50 to 100 tokens per catalog entry, with activated instructions kept below 5,000 tokens where practical.

Tools can use the same structure. Keep a stable namespace such as `orders` or `documents`, describe what lives there, and expose full schemas only after the model or application has selected that surface. OpenAI's [tool search guide](https://developers.openai.com/api/docs/guides/tools-tool-search) calls this deferred loading. It notes that discovered tools can be added at the end of context, preserving the earlier cacheable prefix.

A capability index is not a permission list. It tells the model what exists. The server still decides whether this authenticated caller may use a capability against a particular resource.

## Policy and capability are different

A tool search can find a function. It should not decide which business or safety rule applies. Those decisions need a deterministic resolver with inputs such as resource scope, effective date, account state, or operation type. The resolver should return the selected policy revision and the facts required to validate an action.

This separation prevents a subtle failure. Suppose a model sees a function called `commit_change` and finds it through search. That discovery does not establish that the change is permitted. The gateway should derive tenant and resource scope from trusted server state, validate the proposed operation, and apply an idempotency key before any mutation.

For policy-heavy systems, the useful sequence is:

1. Interpret the request and identify candidate capabilities.
2. Resolve mandatory policy bundles in code.
3. Load the smallest decision cards and schemas needed for this turn.
4. Let the model propose a typed plan.
5. Re-resolve and validate the plan before a side effect.

The model can help with language and ambiguity. It should not be the only component deciding entitlement or authorization.

## Under-disclosure is a real failure

Just-in-time loading can be taken too far. An index that says only `miscellaneous operations` is not an index; it is a locked door. If descriptions omit the inputs, limits, or likely use cases, the model cannot select the right capability. It may guess, ask the user to do work the agent could have done, or use a broader tool because the narrow one was invisible.

A good catalog has enough information to route without carrying the full schema. State what the capability reads or changes, identify the main resource, name important prerequisites, and say when it is not appropriate. Keep descriptions distinct. `get_invoice` and `refund_invoice` should not look like variants of the same vague verb.

Under-disclosure also appears in policy loading. A resolver that returns a policy ID but no applicability summary forces the model to request material it cannot interpret. Return a compact card with scope, required facts, permitted outcomes, prohibitions, and escalation conditions. Keep the authoritative source and full evidence available for a second lookup.

## Keep loading deterministic

If the model can load a reference, the loading mechanism should be observable. Record the capability name, policy revision, resource identifier, and reason for activation. Use stable ordering and stable serialization so an activated bundle does not churn the reusable prefix for unrelated requests.

Do not silently inject the same instruction on every retry. Track which skills or tool schemas are active in the current turn. If a long-running session compacts its context, protect the activated instructions or reload them from the recorded manifest. The Agent Skills guide explicitly warns that pruning active skill content can degrade behavior without an obvious error.

There is also a useful fallback when provider-native deferred tools are unavailable: expose a small, stable dispatcher or a few domain gateways. The gateway can return typed results and the next allowed operation without placing a large changing tool array at the front of every request. This costs some implementation work, but it preserves a clean boundary between discovery and authority.

## Measure the trade

Deferred loading adds a lookup step. That step can improve context size and cost while hurting latency or recall. Evaluate both. Compare task completion, wrong-tool rate, clarification rate, input tokens, cache reads and writes, and time to completion against an eager baseline.

A capability index is successful when the model can find the right tool without carrying every tool's schema. It is not successful merely because the prompt is shorter. If an important operation is missed, the token saving is a false economy.

The practical rule is simple: advertise capabilities broadly enough to discover them, load details narrowly enough to keep context usable, and let code own policy and side effects.
