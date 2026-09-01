# Database/JPA Case — The Endpoint With 1,001 Queries

**Level:** Senior  
**Time box:** 30 minutes  
**Focus:** JPA/Hibernate, query shape, ORM behavior, latency, database load

## System context

A Spring Boot API returns orders with related customer and line-item data.

The endpoint works well in test and staging. Production latency grows sharply as result size increases.

A simplified service looks reasonable:

```java
@Transactional(readOnly = true)
public List<OrderDto> listOrders() {
    return orderRepository.findRecent().stream()
        .map(orderMapper::toDto)
        .toList();
}
```

The mapper accesses relationships while constructing the DTO.

## Production symptoms

For a request returning 500 orders:

- application CPU is not saturated,
- individual SQL queries are fast,
- the database shows a large increase in query count,
- p95/p99 endpoint latency rises disproportionately with result count,
- connection-pool pressure appears during bursts.

An engineer says:

> "The queries are all under a few milliseconds, so the database cannot be the problem."

## Candidate task

Diagnose the performance failure and explain:

1. What data you would gather first.
2. Why many individually fast queries can still create a slow endpoint.
3. Where lazy loading and DTO mapping can hide database access.
4. Which fixes are appropriate for different endpoint shapes.
5. Why `JOIN FETCH` is not a universal answer.
6. How pagination, collection joins, cardinality and memory affect the design.
7. How you would prevent regression.

## Evidence to request

A strong investigation might use:

- SQL/query-count tracing per request,
- Hibernate statistics,
- datasource proxy instrumentation,
- distributed traces with database spans,
- connection-pool wait time,
- rows returned per query,
- query plans for the expensive aggregate query variants,
- request result cardinality,
- JVM allocation/serialization cost after query consolidation.

## Failure model

A naive object traversal can create a pattern similar to:

```text
1 query  → orders
500      → customer or related-object lookups
500      → line-item/other relationship lookups
-------------------------------
~1,001 database round trips
```

The exact count is less important than the scaling behavior.

Even if each query is quick, repeated round trips accumulate:

```text
network latency
+ driver work
+ pool acquisition
+ database scheduling
+ parsing/planning overhead
+ ORM materialization
```

## Candidate design space

Do not jump to one fix. Compare options such as:

### Purpose-built projection

Fetch exactly the columns required by the endpoint.

Useful when the response is read-oriented and stable.

### Entity graph / fetch plan

Control which associations load for this use case.

Useful when keeping entity semantics is valuable.

### Batch fetching

Reduce round trips while avoiding some large join explosions.

Useful for particular relationship shapes.

### Join fetch

Can be effective, but requires care around collection multiplicity, duplicate rows, pagination, and result explosion.

### Two-step query

Page parent IDs first, then fetch the required graph in a second bounded query.

Useful when pagination and to-many relationships collide.

## Solution direction — intentionally incomplete

The senior-level diagnosis is not merely:

> "This is N+1."

It is:

> **The persistence access pattern scales with object traversal rather than the response's intended query plan. We need an explicit, bounded query shape for this endpoint.**

A production-quality answer also explains why the new query shape remains safe at high cardinality.

## Verification

Define acceptance criteria before implementing the fix.

For example:

```text
query count per request: bounded and predictable
p95/p99 latency:         within target at realistic page size
pool wait time:          near zero under expected concurrency
rows materialized:       bounded
heap/allocation:         no new regression from giant object graphs
```

Then add a regression test that fails when query count grows unexpectedly.

## Self-review checklist

- [ ] Measured query count per request
- [ ] Explained why fast queries can still be slow in aggregate
- [ ] Connected lazy relationships to mapping/serialization
- [ ] Offered more than one query-shaping strategy
- [ ] Discussed to-many join explosion
- [ ] Discussed pagination correctness
- [ ] Included connection-pool behavior
- [ ] Included regression protection

## Staff-level follow-up

Your organization has 80 Spring/JPA services. How would you detect N+1 regressions before production without banning ORM usage?

Consider test instrumentation, query budgets, tracing standards, code-review heuristics, and endpoint-specific data-access contracts.

---

Previous: [Spring Transaction Boundary](../spring/transactional-self-invocation.md) · Next: [Kafka Duplicate Side Effect →](../kafka/duplicate-event.md)
