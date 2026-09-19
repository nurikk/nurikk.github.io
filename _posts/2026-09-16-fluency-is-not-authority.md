---
title: Fluency is not authority
date: 2026-09-16 09:00:00 +0100
categories: [AI safety]
tags: [side effects, prompt injection, reasoning traces]
description: Language models are good at interpreting requests. Deterministic software must decide which side effects are allowed, and under which facts.
image:
  path: /assets/img/posts/fluency-is-not-authority.webp
  alt: A flowing language signal reaches a hard authorization gate.
---

A model can give a convincing explanation of an action it is not allowed to take. It can also describe a safe-looking plan that contains an unsafe operation. Fluency is evidence that the model produced coherent language. It is not evidence that the proposed side effect is authorized.

That boundary is easy to lose in an agent loop. The model reads a request, calls a tool, reads the result, and proposes the next call. If the tool gateway treats the model's argument as authority, the language layer has quietly become the permission layer.

## Put authority in code

The model is useful for interpreting language, resolving ordinary ambiguity, asking for missing information, and producing a typed proposal. Code should own authentication, resource scope, eligibility, calculations, approvals, and mutation. A gateway should derive the caller and target resource from trusted server state rather than accepting an arbitrary model-supplied identity.

A safe action path usually has a proposal phase and a commit phase. The proposal contains the intended operation, the policy revisions it used, the relevant facts, and any missing conditions. The server resolves policy again, checks live state, and either commits the narrow operation or returns a reason it cannot proceed.

This is not a claim that deterministic code should replace language understanding. It is a claim about the cost of being wrong. A misunderstood sentence can be repaired in the next turn. An unauthorized transfer, deletion, or disclosure may not be reversible.

## A reasoning blob is not a permission token

The distinction also applies to opaque model state. Some APIs return a visible summary plus a blob that the client must send back to continue a stateless reasoning exchange. The blob may be called a signature, encrypted content, or a thinking block. The client cannot read it, but a valid blob still has to be treated as data with a defined trust boundary.

The public [Stolen Thoughts analysis](https://arxiv.org/html/2608.09867v1) describes a worked boundary failure: a valid reasoning blob from one model interaction was replayed into a compatible model context, and extraction prompts were used to make the model reproduce internal content. The associated [project page](https://stolen-thoughts.com) and the [public disclosure discussion](https://blog.cryptographyengineering.com/2026/05/29/fooling-around-with-encrypted-reasoning-blobs) provide additional context.

The important lesson is not that every current endpoint is exploitable. The analysis covers particular API and model versions and reports that providers changed behavior after disclosure. The lesson is that opacity at the client boundary is not the same as access control. If a server accepts a blob without binding it to the caller, session, model class, and position in a conversation, replay may be broader than its designers intended.

Consider the failure as a sequence:

1. A client obtains a valid opaque reasoning block.
2. Another request supplies that block to a compatible decoder context.
3. The model is instructed to transcribe or reinterpret what the block contains.
4. The resulting text crosses a boundary that assumed the block was unreadable.

At no point does cryptanalysis have to succeed. The API itself supplies a valid decryption or interpretation path. A blob that can be replayed is an input, not a secret store.

## The remediation is a boundary repair

A provider can bind stateless state to an authenticated caller, session, model family, conversation position, and predecessor hash. It can store state server-side behind a random lookup identifier instead of giving the client a reusable blob. It can isolate cross-model replay and detect repeated failures or repeated use of one value. Each option has operational costs, and a hash chain without replay state does not prevent replaying a complete copied conversation.

Applications have work to do as well. Do not publish opaque reasoning fields in raw traces. Removing only visible plaintext is not enough if an opaque field is accepted by a later endpoint. Treat imported agent trajectories as untrusted input. Validate role, session, model, and sequence before continuing them. Keep extraction resistance as defense in depth, not as the primary authorization mechanism.

The same rule governs ordinary tool results. A retrieved document can contain instructions, but it cannot promote itself to a developer message. A model can quote a policy, but it cannot establish that the policy applies. A reasoning trace can contain a plan, but it cannot authorize the plan.

Design reviews should ask one blunt question: what exact component can authorize this side effect, and what evidence does it verify? If the answer is "the model because it saw the right instructions," the boundary is in the wrong place. The model can recommend. The server decides.
