# JVM Case — Leak or Load? Diagnosing Rising Heap

**Level:** Senior+  
**Time box:** 30 minutes  
**Focus:** JVM memory, GC, retained heap, caches, observability, incident judgment

## System context

A Java 21 REST service performs document enrichment. It runs as 12 Kubernetes pods. Each pod has:

```text
Container memory limit: 6 GiB
Max Java heap:          4 GiB
Normal deployment RPS: ~450
Business-hour bursts:  ~1,200 RPS for 10–20 minutes
```

The request path:

```text
HTTP request
   ↓
JSON parsing
   ↓
intermediate domain objects
   ↓
2 downstream calls
   ↓
response enrichment
   ↓
bounded in-process cache
```

A recent release added richer response metadata and changed caching behavior.

## Production symptoms

For the first few hours after deployment, latency is normal. Over 8–12 hours:

- old-generation occupancy trends upward,
- full-GC frequency increases,
- p99 latency rises from ~240 ms to >1.8 s,
- some pods restart after sustained GC pressure,
- restarting a pod immediately restores low heap occupancy and normal latency.

The incident channel already calls this a **memory leak**.

Your job is not to accept that diagnosis without evidence.

## Evidence

Representative pod telemetry:

| Time | RPS | Heap used after GC | Allocation rate | Full GC |
|---|---:|---:|---:|---:|
| 09:00 | 42 | 1.1 GiB | 620 MiB/s | 85 ms |
| 12:00 | 96 | 1.8 GiB | 1.4 GiB/s | 140 ms |
| 15:00 | 88 | 2.5 GiB | 1.2 GiB/s | 310 ms |
| 18:00 | 51 | 2.7 GiB | 690 MiB/s | 520 ms |

GC logs show heavy young-generation reclamation, but the post-GC old-generation floor rises through the day.

A late-day heap histogram contains large populations of:

```text
byte[]
java.lang.String
MetadataValue
CacheEntry
```

The cache is configured for a nominal maximum of **50,000 keys**.

## Candidate task

Explain:

1. Why rising heap is not enough to prove a memory leak.
2. The plausible causes you would rank first.
3. What evidence would distinguish transient allocation pressure, a larger legitimate working set, and unbounded retention.
4. Your investigation sequence during the incident.
5. A safe mitigation you could apply before the full root cause is known.
6. What a durable fix could look like.
7. Exactly how you would verify the repair.

## Clarifying questions worth asking

Try to generate your own before revealing this list.

<details>
<summary>Show prompts</summary>

- Does the post-full-GC heap return to a stable baseline after traffic falls?
- Do we have multiple heap dumps taken at comparable points in time?
- Which objects dominate **retained size**, not just shallow size?
- Is the cache bounded by entry count, memory weight, TTL, or some combination?
- Did average cached value size change in the release?
- Does cache key cardinality continue to grow?
- Is native/off-heap memory growing too?
- Which collector and heap/container settings are in use?
- What changed in allocation rate after the release?

</details>

## Interviewer injects

Ask a colleague to reveal these one at a time only when your investigation justifies them:

<details>
<summary>Inject A</summary>

A heap dump shortly after startup contains roughly 8,000 cache entries. Ten hours later the cache is close to its configured maximum.

</details>

<details>
<summary>Inject B</summary>

Retained-size analysis shows the cache entry graph dominates the live set.

</details>

<details>
<summary>Inject C</summary>

The cache is bounded by **entry count**, not object weight. The recent feature materially increased the average cached value size.

</details>

<details>
<summary>Inject D</summary>

When traffic drops, post-GC heap stabilizes at a high floor rather than continuing to rise indefinitely.

</details>

## Solution direction — intentionally incomplete

A senior candidate should distinguish:

```text
allocation rate ≠ retained live set ≠ unbounded leak
```

The important question is not simply *"Is heap high?"* but:

> **What owns the live objects after GC, how does that ownership change over time, and does the live set converge to a stable working set?**

A bounded cache can still be dangerously large if the bound is expressed only as **number of entries** while the average object graph becomes much larger.

Your production answer should include both:

- an incident action that restores memory headroom without accidentally overwhelming downstream systems, and
- a durable memory-budgeting strategy that is verifiable with production-like key cardinality and traffic bursts.

## Self-review checklist

Give yourself one point for each item you covered convincingly:

- [ ] Refused to declare a leak from heap growth alone
- [ ] Separated allocation pressure from retained memory
- [ ] Asked for post-GC/live-set behavior
- [ ] Used retained-size/dominator analysis, not just class counts
- [ ] Challenged the assumption that entry-count bounds memory
- [ ] Considered cache/downstream trade-offs before mitigation
- [ ] Defined measurable verification criteria
- [ ] Included a long-running soak test
- [ ] Included production-like key cardinality
- [ ] Included GC, p99, cache and downstream metrics

**8–10:** strong senior signal  
**6–7:** reasonable, but sharpen production verification  
**<6:** revisit memory diagnosis before moving on

## Staff-level follow-up

Your platform has 40 Java services with local caches. How would you prevent this class of failure organizationally rather than fixing it service by service?

Think about:

- memory budgets,
- cache libraries/defaults,
- telemetry standards,
- load/soak gates,
- container/heap policy,
- incident playbooks.

---

Next drill: [Transactional Self-Invocation →](../spring/transactional-self-invocation.md)
