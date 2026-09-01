# Spring Case — Transactional Self-Invocation

**Level:** Senior  
**Time box:** 25 minutes  
**Focus:** Spring AOP, proxy boundaries, transaction semantics, business invariants

## System context

A Spring Boot service updates an account and writes an audit record.

The team expects the database changes to commit or roll back as one unit.

Simplified code:

```java
@Service
public class AccountService {

    private final AccountRepository accounts;
    private final AuditRepository audits;

    public void updateAccount(UpdateCommand command) {
        validate(command);
        persistChange(command);
    }

    @Transactional
    public void persistChange(UpdateCommand command) {
        Account account = accounts.findById(command.accountId())
            .orElseThrow();

        account.apply(command);
        accounts.save(account);

        audits.save(AuditEntry.forUpdate(command));
    }
}
```

Production reports show paths where behavior does not match the team's expected transaction boundary.

An engineer responds:

> "But `persistChange()` has `@Transactional`."

## Candidate task

Explain:

1. Why the annotation may not create the transaction the engineer expects.
2. What runtime mechanism Spring normally uses to apply transactional interception.
3. How you would prove the hypothesis rather than merely recognizing the pattern.
4. The safest short-term repair.
5. The cleaner production design.
6. Which business boundary should own the transaction.
7. When simply adding more annotations makes the design worse rather than safer.

## Investigative prompts

Before giving the diagnosis, ask:

- Is `updateAccount()` invoked through a Spring-managed bean reference?
- Is `persistChange()` called from the same instance?
- Which transaction-management mode is configured?
- What do transaction logs or `TransactionSynchronizationManager` show at runtime?
- Do repository methods happen to create smaller independent transactional boundaries?
- Which database writes must be atomic from the business perspective?
- Is any remote I/O being performed inside the desired transaction?

## The trap

This code looks declarative, but annotations are not magic syntax handled by the JVM.

For normal proxy-based Spring transaction management, interception happens when a call **crosses the proxy boundary**.

Think carefully about this call path:

```text
external caller
   ↓
Spring proxy
   ↓
updateAccount()
   ↓
this.persistChange()
```

Which arrow crosses the transactional interceptor?

## Solution direction — intentionally incomplete

The key production lesson is broader than "self-invocation breaks `@Transactional`."

A strong answer should say:

> **Place the transaction around the actual business unit of work, not around a private implementation detail you happen to want intercepted.**

Possible repairs include moving the transaction boundary to an externally invoked orchestration method or extracting the transactional component into another Spring-managed bean when that separation is architecturally meaningful.

Do not stop at syntax. Explain the atomicity requirement you are protecting.

## Verification challenge

How would you prove the repaired implementation actually rolls back correctly?

Design tests for at least these failure points:

```text
1. account loaded
2. account changed
3. account row written
4. audit write attempted
5. audit write fails
```

What database state should remain after step 5 fails?

Then add:

- concurrent updates,
- retry behavior,
- deadlock/serialization exceptions,
- connection loss near commit.

## Weak-answer warning signs

- "Add `@Transactional` again."
- "Make the method public."
- Memorizes self-invocation but cannot explain the proxy call path.
- Cannot state the business transaction boundary.
- Ignores verification/rollback testing.
- Pulls remote calls into a long database transaction without discussing consequences.

## Self-review checklist

- [ ] Explained proxy-based interception
- [ ] Identified the same-instance call path
- [ ] Defined the business atomicity boundary
- [ ] Proposed a safe repair, not annotation cargo-culting
- [ ] Explained how to verify rollback behavior
- [ ] Considered transaction duration and remote I/O
- [ ] Discussed concurrency/failure behavior

## Staff-level follow-up

Suppose the audit requirement changes: audit events must now be consumed by another service and survive broker outages.

Would you keep both writes in one local transaction? Where would a transactional outbox fit, and which guarantees would it actually provide?

---

Previous: [JVM Rising Heap](../jvm/rising-heap-debugging.md) · Next: [N+1 Diagnosis →](../database/n-plus-one-diagnosis.md)
