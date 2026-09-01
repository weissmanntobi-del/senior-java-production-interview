# System Design Case — Payment Processing Service

**Level:** Staff-leaning  
**Time box:** 60 minutes  
**Focus:** idempotency, state machines, ledgers, provider ambiguity, outbox, reconciliation, scaling, security

## Prompt

Design a high-value payment-processing platform built with Java services.

It must support:

- card and bank-payment providers,
- duplicate client retries,
- provider timeouts,
- delayed provider callbacks,
- partial failures,
- refunds,
- reconciliation,
- auditability,
- horizontal scaling,
- secure handling of payment metadata.

Representative workload:

```text
Normal traffic:       3,000 payment attempts/sec
Peak traffic:        12,000 attempts/sec
Availability target: 99.99%
Multi-region clients: yes
Provider latency:     100 ms to several seconds
```

A client can retry if its original response is lost.

## Failure conditions your design must survive

- The same payment request arrives multiple times.
- The provider succeeds but your service times out.
- Your database commits but event publication is delayed.
- Provider callbacks arrive more than once or out of order.
- A payment remains in an ambiguous state.
- A process restarts during execution.
- Reconciliation later finds provider state different from local state.

## Ambiguous timeline

```text
12:00:00.000  POST /payments idempotencyKey=K-100
12:00:00.080  local payment record created
12:00:00.150  provider authorization submitted
12:00:02.500  client timeout
12:00:02.900  provider completes authorization
12:00:03.000  service process crashes
12:00:03.200  client retries with K-100
```

Provider capabilities:

```text
provider idempotency key
provider transaction lookup
asynchronous signed webhook callbacks
```

## Candidate task

Design and defend:

1. API contract
2. payment/payment-attempt identity
3. idempotency contract
4. request fingerprinting
5. payment state machine
6. database model
7. financial ledger model
8. provider adapters
9. timeout/unknown-outcome handling
10. callback handling
11. reconciliation
12. event publication
13. consistency boundaries
14. horizontal scaling
15. multi-region approach
16. observability
17. security and audit controls
18. important trade-offs

## Discovery questions

Before drawing architecture, clarify:

- Is an idempotency key scoped by merchant, payment intent, or payment attempt?
- Can one business order have multiple legitimate attempts?
- How long must dedupe records survive?
- What are legal states visible to clients?
- Is an append-only financial ledger required?
- Can provider authorization always be looked up?
- Are webhooks authenticated/signed?
- Which fields enter PCI scope?
- Are raw PANs ever persisted?
- Which guarantees must hold within one region vs globally?
- How quickly must unknown outcomes be reconciled?

## Architecture whiteboard

Start with components, but do not mistake components for a correctness model.

```text
Clients
   ↓
API Gateway
   ↓
Payment API
   ↓
Payment Orchestrator
   ├────────────→ Provider Adapter → External Provider
   │
   ↓
PostgreSQL
   ├─ payment_intent
   ├─ payment_attempt
   ├─ idempotency_record
   ├─ ledger_entry
   └─ outbox_event

Outbox Publisher → Kafka → Order / Notification / Risk consumers

Reconciliation Service → Provider Lookup APIs

Provider Webhook → Webhook Ingress → Payment state transition
```

Now define which arrows are synchronous, which are retryable, and which boundaries are atomic.

## State-model exercise

Create an explicit state machine. At minimum, consider why a system might need a state such as:

```text
PENDING_PROVIDER_CONFIRMATION
```

rather than forcing every timeout into `FAILED`.

Questions:

- What transition occurs when the provider call times out?
- What if provider lookup later says `AUTHORIZED`?
- What if a duplicate webhook arrives?
- What if client retry arrives while provider outcome is unknown?
- Which transitions are monotonic?
- Which operations require new identities (for example refunds)?

## Idempotency contract

Define exact behavior for:

```text
same key + same payload
same key + different payload
same key + request currently in progress
same key + original response lost
same key + operation already completed
```

A good design binds the idempotency key to enough request identity to detect semantic conflicts rather than blindly replaying unrelated operations.

## Ledger challenge

Explain the difference between:

```text
payment workflow state
```

and

```text
financial ledger entries
```

Why might one be mutable while the other should be append-oriented/auditable?

What invariants must always hold?

## Event-publication challenge

Suppose PostgreSQL commits payment state, then the process dies before publishing `PaymentAuthorized`.

How does the event recover?

Your answer should name the local transaction boundary and why downstream consumers still need idempotency under at-least-once event delivery.

## Provider timeout challenge

Never answer a timeout with:

> "The payment failed."

unless the provider contract gives you evidence of failure.

A timeout can mean:

```text
request never arrived
request arrived and failed
request succeeded but response was lost
provider is still processing
```

Design for ambiguity rather than pretending the network can tell you which occurred.

## Multi-region extension

Your API is active-active in two regions.

Two requests with the same idempotency key arrive almost simultaneously in different regions.

How do you prevent two provider attempts?

Compare at least two approaches and their trade-offs:

- single home region per merchant/payment key,
- globally coordinated key ownership,
- database/global consistency mechanisms,
- routing/partitioning strategies.

Discuss latency, availability, failover and split-brain risk.

## Observability scorecard

Your design should make these measurable:

- payment state-transition latency,
- provider latency/error rate by provider,
- number/age of unknown payments,
- reconciliation backlog and mismatch rate,
- duplicate requests suppressed,
- duplicate callbacks suppressed,
- outbox lag,
- stuck state counts,
- ledger invariant violations,
- refund failure/retry rate,
- per-provider circuit/rate-limit state.

## Security checklist

Cover:

- provider/tokenization strategy to reduce raw-card exposure,
- encryption in transit and at rest,
- secret/key management,
- signed callback validation,
- authorization for payment/refund operations,
- least-privilege service credentials,
- audit access,
- sensitive-field redaction,
- operational break-glass controls.

## Weak-answer signals

- "Use exactly-once."
- "Retry the provider on timeout."
- "Put `@Transactional` around the whole payment."
- One mutable `payment.status` field with no durable history.
- No idempotency contract.
- No explicit ambiguous state.
- No reconciliation path.
- No answer for duplicate callbacks.
- No event-publication recovery.
- No security boundary.

## Self-evaluation rubric

Score each dimension 0–5.

| Dimension | 0 | 3 | 5 |
|---|---|---|---|
| Idempotency | Missing | Stable key + local dedupe | Explicit contract + conflict detection + provider idempotency |
| Failure reasoning | Happy path only | Handles retries/timeouts | Models crash windows, ambiguity, callbacks and reconciliation |
| Data model | One status row | Durable workflow state | Workflow + append-oriented ledger + audit invariants |
| Events | Fire-and-forget | Recoverable publishing | Outbox + idempotent consumers + replay reasoning |
| Provider integration | Direct call | Adapter + retry controls | Provider-specific idempotency, lookup, isolation and reconciliation |
| Scalability | Stateless workers | Capacity-aware design | Peak/load isolation + multi-region correctness trade-offs |
| Observability | Logs | Useful service metrics | State/invariant/reconciliation observability |
| Security | Generic encryption | Tokenization + access control | Explicit PCI-scope reduction, keys, callbacks, audit and operations |

**32–40:** strong Staff-leaning response  
**24–31:** solid Senior response  
**16–23:** architecture is plausible but production correctness is incomplete  
**<16:** revisit failure-state modeling before optimizing components

## Interviewer's final question

> What is the single most important invariant in this architecture, and which mechanism actually enforces it?

If your answer is a product name rather than a business invariant, keep working.

---

Previous: [Kafka Duplicate Side Effect](../kafka/duplicate-event.md) · [Explore the full 50-scenario pack →](../full-pack.md)
