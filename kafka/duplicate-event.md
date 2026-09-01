# Kafka Case — Duplicate Event, Duplicate Side Effect

**Level:** Senior+  
**Time box:** 30 minutes  
**Focus:** Kafka delivery semantics, idempotency, offsets, external side effects, crash recovery

## System context

A Java Kafka consumer processes a business event and calls an external HTTP service that performs a non-trivial side effect.

Simplified flow:

```text
Kafka record
   ↓
validate
   ↓
HTTP side effect
   ↓
mark processing complete
   ↓
commit Kafka offset
```

The system uses at-least-once delivery.

## Incident

A customer receives the business side effect twice.

Logs show the same event ID was processed in two separate consumer attempts around a rebalance/crash window.

The team says:

> "Kafka delivered a duplicate. We should enable exactly-once."

## Candidate task

Explain:

1. How the duplicate business effect can occur even if Kafka is functioning correctly.
2. Which failure window is most dangerous.
3. What Kafka offset commits do and do not prove.
4. Why Kafka exactly-once semantics do not automatically make an arbitrary external HTTP API exactly-once.
5. How you would redesign the consumer for business-level duplicate safety.
6. What key/idempotency state you would persist.
7. What happens during retries, rebalances, process crashes, and timeouts.
8. How you would test the design.

## Failure timeline

Reason through this sequence:

```text
T0  record received: eventId=E-9007
T1  external HTTP request succeeds
T2  consumer process crashes
T3  offset was not durably committed
T4  partition assigned again
T5  E-9007 is delivered again
T6  external HTTP request runs again
```

Where is the atomic boundary?

## Clarifying questions

- Does the downstream HTTP API support an idempotency key?
- Can the downstream state be queried after a timeout?
- Is the event ID globally stable and unique for the business operation?
- Do we maintain a processed-event/inbox table?
- Is the side effect reversible?
- What are the consequences of duplicate vs lost processing?
- Are Kafka transactions used anywhere, and what resources are actually inside that transaction?
- Can processing be safely retried indefinitely?

## Core distinction

Interviewers often want you to say this precisely:

```text
message delivery semantics
        ≠
business effect semantics
```

And similarly:

```text
Kafka transaction
        ≠
atomic transaction across Kafka + arbitrary HTTP service
```

## Solution direction — intentionally incomplete

A durable answer starts from the **business operation identity**.

Depending on downstream capability, the design may use:

- a stable idempotency key passed to the external service,
- a local inbox/processed-event record protected by a unique constraint,
- a durable operation state machine,
- reconciliation/lookup for ambiguous remote outcomes,
- compensation when reversibility exists.

The exact pattern depends on where the irreversible side effect occurs.

Do not claim a stronger guarantee than the system actually provides.

## Dangerous answer

> "Commit the offset before the HTTP call so the event cannot repeat."

That converts duplicate risk into **loss risk** if the process dies after the commit but before the side effect.

A senior engineer should explicitly compare those failure modes.

## Verification drills

Inject failures at every boundary:

```text
before HTTP request
HTTP timeout with unknown remote outcome
after HTTP success
before local dedupe commit
after local state commit
before Kafka offset commit
consumer rebalance
process kill -9
network partition
```

For each injection, answer:

1. What state is durable?
2. Will the record be retried?
3. Could the side effect repeat?
4. How does the system converge?

## Self-review checklist

- [ ] Identified the crash-after-side-effect/before-offset window
- [ ] Distinguished Kafka correctness from business idempotency
- [ ] Rejected blanket "exactly once" claims
- [ ] Used a stable business operation ID
- [ ] Addressed ambiguous HTTP timeouts
- [ ] Covered retries and rebalances
- [ ] Defined durable deduplication/reconciliation state
- [ ] Included failure-injection verification

## Staff-level follow-up

Design a shared platform pattern that 30 consumer teams could use for external side effects without every team inventing deduplication independently.

What belongs in a library, a service, a database convention, an API contract, or an operational standard?

---

Previous: [N+1 Diagnosis](../database/n-plus-one-diagnosis.md) · Next: [Payment System Design →](../system-design/payment-service-prompts.md)
