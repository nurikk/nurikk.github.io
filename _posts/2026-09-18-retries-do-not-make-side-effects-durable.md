---
title: Retries do not make side effects durable
date: 2026-09-18 09:00:00 +0100
categories: [Reliable systems]
tags: [retries, idempotency, durable execution]
description: A retry can recover from an uncertain response only when the operation has replay-safe semantics and the durable state proves what happened.
image:
  path: /assets/img/posts/retries-do-not-make-side-effects-durable.webp
  alt: A workflow crosses checkpoints while a lock prevents duplicate work.
---

A timeout after a write is not the same as a failed write. The server may have committed the change while the client was waiting. Retrying the request can therefore produce a duplicate, not a recovery.

This is the awkward middle state that reliable systems must model: the caller does not know whether the side effect happened. A retry policy alone cannot resolve that uncertainty.

## Replay is a normal execution path

Durable workers replay after a process crash. Queues redeliver after an acknowledgement timeout. HTTP clients retry after a connection reset. These are not unusual edge cases. If an operation can change state, replay belongs in its design and test suite.

A checkpoint records enough durable progress to resume without repeating completed work. It must be written at a useful boundary, not only in process memory. For a multi-step agent run, that might mean recording the validated action proposal before execution, the provider's operation identifier after acceptance, and the observed committed state before moving to the next step.

A checkpoint is not a magic duplicate filter. If it says only `started`, recovery still needs to know whether the external system accepted the request. Store an external operation ID or a result that can be queried, and make the recovery path explicit for unknown outcomes.

## Give mutations an idempotency key

An idempotency key ties retries of one logical operation to one durable result. The key must be generated for the operation, persisted with its intent, and reused when the caller retries. The server stores the key with enough request identity and result data to reject a conflicting reuse and return the original outcome for an equivalent replay.

The [AWS reliability guidance](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_prevent_interaction_failure_idempotent.html) recommends making mutating operations idempotent and checking whether a unique identifier has already been processed. [Stripe's API documentation](https://docs.stripe.com/api/idempotent_requests) is a concrete public example: a repeated request with the same key can return the prior result instead of creating a second operation.

The key is not a substitute for authorization or validation. Validate the current caller, resource scope, request shape, and policy before looking up or creating the idempotency record. Decide how long records live, what happens after expiry, and whether the same key with different parameters is an error. Those are part of the API contract.

## Do not publish an event from memory

A common failure occurs when a service commits a database change and then publishes an event in a separate step. A crash between the two leaves durable state without notification. Reversing the order creates the opposite problem: consumers receive an event for a change that never committed.

The [transactional outbox pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html) addresses this by writing the business change and an outbox record in one database transaction. A separate publisher reads committed outbox records and sends them to the message system. The publisher can retry. Consumers still need idempotent handling because delivery can repeat.

The outbox is a publication checkpoint, not a guarantee that every downstream consumer succeeds. Include a stable event ID, aggregate identifier, version, and enough data for consumers to process the event without guessing. Track attempts and dead-letter events that need intervention. If a consumer applies a state change, give that consumer its own deduplication boundary.

## A safe agent action loop

A side-effecting agent can use a narrow sequence:

1. Build a typed proposal from the request and current facts.
2. Validate authorization, policy revision, and live state.
3. Persist the proposal and idempotency key.
4. Execute the external mutation with that key.
5. Check the external result or query the operation status after an uncertain response.
6. Commit the local state and outbox event atomically.
7. Publish the event and make consumer replay safe.

A retry may happen at steps four, five, or seven. The retry must use the same identity and the same recovery rule. It should not invent a new operation because the previous response was lost.

Checkpoints also need versioning. If a worker resumes with a changed tool contract or policy revision, blindly continuing can apply a stale plan. Store the versions used to create the proposal and invalidate or revalidate it when the authoritative state changes.

Reliability comes from making the uncertain states explicit. “The request timed out” is not a final business result. “The request with key K is committed as operation O” is. Until the system can establish that fact, it should recover by querying, not by guessing.
