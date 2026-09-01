# Senior Java Production Interview

> Production-grade interview drills for senior Java backend engineers — focused on diagnosis, failure modes, distributed systems, and architecture rather than trivia.

[![Java](https://img.shields.io/badge/Java-21+-orange)](#)
[![Level](https://img.shields.io/badge/Level-Senior%20%2F%20Staff-blue)](#)
[![Focus](https://img.shields.io/badge/Focus-Production%20Reasoning-success)](#)
[![Scenarios](https://img.shields.io/badge/Full%20Pack-50%20Scenarios-purple)](full-pack.md)

Explore the Senior Java Production Interview System → https://trivajay.gumroad.com/l/nflnlb

## Why this repository exists

Most Java interview preparation teaches you to answer questions such as:

- What is the difference between `HashMap` and `ConcurrentHashMap`?
- What does `@Transactional` do?
- What is an N+1 query?
- What does Kafka's at-least-once delivery mean?

Senior interviews often ask a harder question:

**Can you recognize those concepts when a production system is failing, reason from incomplete evidence, choose a safe mitigation, and design a durable fix?**

This repository gives you five free interview-style production drills extracted and adapted from the **Senior Java Production Engineering Interview System**.

## Free production drills

| Domain | Scenario | Core signal | Suggested time |
|---|---|---|---:|
| JVM | [Leak or Load? Diagnosing Rising Heap](jvm/rising-heap-debugging.md) | Retention vs allocation pressure | 30 min |
| Spring | [Transactional Self-Invocation](spring/transactional-self-invocation.md) | Proxy boundaries and transaction semantics | 25 min |
| Database/JPA | [The Endpoint With 1,001 Queries](database/n-plus-one-diagnosis.md) | ORM convenience vs query shape | 30 min |
| Kafka | [Duplicate Event, Duplicate Side Effect](kafka/duplicate-event.md) | Delivery semantics vs business idempotency | 30 min |
| System Design | [Payment Processing Service](system-design/payment-service-prompts.md) | Durable idempotent workflow design | 60 min |

## How to use each case

Do not read ahead looking for the answer. Treat each file as a mock interview.

1. Set the recommended timer.
2. Read only **System Context**, **Symptoms**, and **Evidence**.
3. State your assumptions and ask clarifying questions out loud.
4. Diagnose the problem before proposing technology.
5. Separate **incident mitigation** from **durable production repair**.
6. Define how you would verify the repair.
7. Compare your answer with the **solution direction** and **self-review checklist**.

A strong senior answer usually follows this shape:

```text
symptoms
   ↓
possible failure modes
   ↓
evidence needed to discriminate
   ↓
most likely cause
   ↓
safe mitigation
   ↓
durable repair
   ↓
verification / rollback / observability
```

## What senior interviewers are evaluating

The cases are intentionally designed to expose more than API knowledge.

| Competency | What strong candidates demonstrate |
|---|---|
| Diagnosis | Distinguish symptoms from causes and request discriminating evidence |
| Java/JVM depth | Connect runtime behavior to memory, concurrency, proxies, pools, and execution boundaries |
| Correctness | Protect business invariants under retries, crashes, duplicates, and races |
| Performance | Reason about bottlenecks with measurements rather than intuition |
| Reliability | Design explicit timeouts, retries, idempotency, backpressure, and recovery |
| Observability | Name the metrics/traces/logs that prove or falsify the hypothesis |
| Production judgment | Separate emergency containment from long-term architecture |
| Communication | Make assumptions and trade-offs explicit |

## Interview rule: don't jump straight to the fix

A weak answer often starts with:

> "Increase the heap."

> "Add `@Transactional`."

> "Use a join fetch."

> "Enable exactly-once."

A stronger answer starts with:

> "Here are the hypotheses, and here is the evidence I need to distinguish them."

That difference is the purpose of this repository.

## Full 50-scenario system

The complete pack expands this approach across:

- JVM and concurrency
- Spring Boot
- PostgreSQL and JPA
- Microservices and resilience
- Kafka and event-driven systems
- Redis and distributed caching
- Production observability
- Java system design

The paid edition includes deeper interviewer material, full reasoning paths, root causes, incident mitigations, durable repairs, verification plans, follow-up questions, staff-level extensions, and scoring rubrics.

See **[full-pack.md](full-pack.md)** for the complete scope.

Explore the Senior Java Production Interview System → https://trivajay.gumroad.com/l/nflnlb

## Who this is for

Best suited to engineers preparing for:

- Senior Java Engineer
- Senior Backend Engineer
- Lead Backend Engineer
- Staff Software Engineer
- Principal-level backend/system-design interviews

It is especially useful if your interviews include Java/Spring depth, production debugging, distributed systems, Kafka, database performance, reliability, or system design.

## Repository philosophy

This is not a leaked-question repository and not a collection of company-specific interview questions. The scenarios are original production-style simulations intended to train transferable engineering reasoning.

---

**Author:** Tobias Weissmann  
**Full pack:** Add your Gumroad/product URL to `full-pack.md` before publishing.
