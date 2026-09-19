---
title: Prompt caching is not memory
date: 2026-09-14 09:00:00 +0100
categories: [AI systems]
tags: [prompt caching, context engineering, model APIs]
description: Prompt caching reuses a stable prefix. It does not remember a conversation, select policy, or shrink a growing context.
image:
  path: /assets/img/posts/prompt-caching-is-not-memory.webp
  alt: A stable token prefix meets a branching conversation trail.
---

Prompt caching is easy to describe badly. A later request can reuse work from an earlier request when the beginning of the rendered prompt is unchanged. That is useful, sometimes very useful, but it is not conversation memory.

The distinction matters because the two features have different owners. A cache belongs to the model-serving path. Memory belongs to the application that decides what facts, events, and unresolved questions should survive a turn.

## What a cache actually holds

The provider receives a sequence of rendered input tokens. It processes that sequence into intermediate state and may retain the state for a prefix. If a later request begins with the same eligible tokens, the provider can resume from the retained state instead of doing all of that prefix work again.

That is the basic contract described in the [OpenAI prompt caching guide](https://developers.openai.com/api/docs/guides/prompt-caching). The [Anthropic documentation](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) describes the same shape with explicit cache breakpoints. The cache is about repeated computation over the same beginning of a request. The model still processes the new suffix and generates a new response.

A useful request layout looks like this:

```text
stable tools and developer instructions
cache breakpoint
current session state
retrieved policy
latest user message
```

The exact API representation varies, but the design question does not. Put content that changes slowly before content that changes often. Keep serialization, tool order, and whitespace deterministic. Do not put a timestamp, request ID, or current user message in the reusable part.

This is an economics decision. A long stable prefix may be cheaper and faster to process after its first use. It can still be too large, confusing, or irrelevant. Caching does not make those tokens disappear from the model's context, and it does not make a large prompt easier for a model to use reliably.

## Invalidation is about bytes, not intent

Two prompts can mean the same thing and still miss the cache if their rendered prefixes differ. Changing a tool description, the order of tools, a developer instruction, a schema, or a serialization detail can change the prefix identity. Compaction can change it too, even when the conversation is logically equivalent.

This gives cache invalidation a practical shape. Version the stable prompt family deliberately. Treat a change to policy language or an output schema as a new cache generation when that is the honest boundary. Measure cache reads, cache writes, uncached input, time to first token, and total cost from provider usage. A guessed hit rate is not an accounting system.

A cache write is not automatically a win. If a request writes a large prefix and no comparable request follows before expiry, the write added work. If every intent change alters the standing instructions, the system may keep writing new prefixes rather than reusing one. Stable prefixes are useful only when they are actually stable and are used often enough to pay back their setup cost.

## Context still grows

Caching does not compact a transcript. If the application appends every user message, tool result, policy packet, and model response, the effective context can keep growing while the first part is cached. The provider may process the prefix cheaply, but the model still has to attend to the context supplied for the request, subject to its own limits and behavior.

That is why history ownership should stay in application state. Store durable facts, decisions, tool outcomes, and open questions in typed records. Keep a bounded transcript for language continuity. Rebuild each request from that state instead of treating a provider response chain as the canonical database.

The same rule applies to policy. A selected policy packet is current-turn context, not necessarily part of the permanent conversation. When a policy changes, the resolver should select a new version and the validator should check it again. A cache hit is never a reason to retain stale authority.

## Memory answers a different question

Memory asks: what should the system know on the next turn, and why should it trust that information? Caching asks: which already-processed prefix can the provider reuse? One needs provenance, scope, retention, correction, and deletion rules. The other needs prefix identity, expiry, and usage accounting.

Confusing them produces familiar failures. A team may keep a giant system prompt because it is cached, even though most of it is irrelevant. It may assume `previous_response_id` makes old instructions permanent, even though the provider chain and application state have different lifecycles. Or it may treat a cache hit as evidence that the current policy was selected correctly. None of those conclusions follows from caching.

The better approach is modest: make the standing prefix small and deterministic, put changing material after the boundary, and give memory its own reducer and tests. For large tool catalogs, defer schemas and load only the capability needed for the current task. That is a separate design problem, covered in [Load tools when you need them]({% post_url 2026-09-15-load-tools-when-you-need-them %}).

Prompt caching is a transport optimization. Use it to control repeated input work. Do not ask it to be a memory system, a policy engine, or a correctness argument.
