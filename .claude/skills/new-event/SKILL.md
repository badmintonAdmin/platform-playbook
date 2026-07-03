---
name: new-event
description: Design a domain event for the broker — the contract (schema, naming, version), publishing via Transactional Outbox, and an idempotent consumer. Use when the user asks to publish/consume an event, add broker/queue messaging, integrate domains via events ("event", "queue", "broker", "consumer", "publish event").
---

# New domain event

Contract: [rules/PLATFORM.md §10](../../../rules/PLATFORM.md);
reliability: [rules/BACKEND_RULES_PRODUCTION.md](../../../rules/BACKEND_RULES_PRODUCTION.md) §9, §11, §20.

## Step 0 — check that an event is appropriate
An event is a **fact that has already happened**, not a command (rule 71). If you need a
request-response or a synchronous operation, that is a call to a public protocol via DI, not an
event. If the producer wants to "tell the consumer what to do," the contract has been designed
incorrectly.

## Step 1 — the event contract
- **Name:** `<domain>.<entity>.<action>` in the past tense: `orders.order.created`.
- **Payload:** a Pydantic schema in the producer's `apps/<domain>/events.py` — this is the
  domain's public contract. Put facts in the payload (id + the data consumers need), not an ORM
  dump.
- **Version:** `eventVersion` in the envelope. Evolution is additive only (new optional
  fields); a breaking change → a new name/version (`...created.v2`), with the old one living on
  as long as consumers exist.
- **Envelope:** `messageId` (UUID), `eventName`, `eventVersion`, `occurredAt` (UTC),
  `correlationId` (propagated from the current request, rule 15).

## Step 2 — publishing (producer)
- **Only via the Transactional Outbox** (rule 75): write the event to the `outbox` table in
  THE SAME transaction (via the UoW) as the domain change. A direct publish to the broker from
  a use case (dual-write) is forbidden.
- A relay/poller reads the outbox and publishes to the broker, marking what has been sent.

## Step 3 — consuming (consumer)
- The consumer depends **only on the event schema** (rule 74), not on the producer's internals.
- **Idempotency is mandatory** (rule 76): dedup by `messageId` (an inbox table) — delivery is
  at-least-once, so retries will happen.
- The handler is a "controller": it pulls a use case from DI (REQUEST scope per message,
  rule 77) and invokes it. There is no business logic in the handler.
- Errors: retry with backoff; a "poison" message after N attempts goes to the DLQ, not into an
  infinite loop.

## Step 4 — tests
- Event schema: serialization/deserialization, envelope complete.
- Producer: the domain change and the outbox write are atomic (a failure between them is
  impossible).
- Consumer: redelivery of the same `messageId` produces no duplicates.

## Verification
- [ ] Name in the past tense, payload is a versionable schema in `events.py`
- [ ] Publishing via the outbox in a single transaction; no dual-write
- [ ] Consumer is idempotent, DLQ is configured, scope is per message
- [ ] Cross-check against the "Events and broker" and "Background work" sections of
      [rules/RULES_CHECKLIST.md](../../../rules/RULES_CHECKLIST.md)
