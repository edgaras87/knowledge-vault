---
title: "@Transactional"
date: 2025-10-22
tags:
  - spring
  - java
  - transactions
summary: Comprehensive guide to using @Transactional in Spring and Jakarta/JTA, focusing on rollback rules, propagation behaviors, and read-only patterns with practical examples.
aliases:
  - Spring @Transactional Annotation
---

# 🧾 Spring `@Transactional` — Rollback Rules, Propagation, and Read-Only Patterns

> A hands-on reference for **Spring and Jakarta/JTA** transaction management.  
> Focuses on what actually bites developers in production: **rollback semantics**, **propagation chains**, and **read-only optimizations**.

---

## 🧩 1. The essence of `@Transactional`

Transactions wrap a series of DB operations into an **atomic unit** — all succeed or all fail.

```java
@Transactional
public void process() {
  repo.save(...);
  repo.update(...);
  if (somethingWrong()) throw new RuntimeException();
}
```

If an unchecked exception occurs, Spring marks the transaction for **rollback**.

---

### Default rules

| Type of Exception        | Spring Default | Jakarta Default | Fix / Override                                                                       |
| ------------------------ | -------------- | --------------- | ------------------------------------------------------------------------------------ |
| RuntimeException / Error | rollback       | rollback        | —                                                                                    |
| Checked Exception        | **commit**     | **commit**      | add `rollbackFor=Exception.class` (Spring) or `rollbackOn=Exception.class` (Jakarta) |
| Caught exception         | commit         | commit          | call `setRollbackOnly()` manually                                                    |

---

## ⚙️ 2. Rollback in practice

### A) Unchecked exception → rollback automatically

```java
@Transactional
public void placeOrder(Long userId) {
  orderRepo.save(new Order(...));
  paymentRepo.save(new Payment(...));
  throw new IllegalStateException("payment gateway down"); // rollback
}
```

Both inserts roll back.

---

### B) Checked exception → commits unless configured

```java
@Transactional
public void placeOrderChecked(Long userId) throws IOException {
  orderRepo.save(...);
  throw new IOException("printer failed"); // commits
}
```

---

### C) Fix: include checked in rollback

```java
@Transactional(rollbackFor = Exception.class)
public void placeOrderCheckedRollback(Long userId) throws IOException {
  orderRepo.save(...);
  throw new IOException("printer failed"); // rolls back
}
```

---

### D) Jakarta equivalent

```java
import jakarta.transaction.Transactional;

@Transactional(rollbackOn = Exception.class)
public void placeOrderJakarta(Long userId) throws IOException { ... }
```

---

## 🔄 3. Propagation behavior

Each transaction can join, suspend, or create a new one.

| Propagation        | Meaning                                | Typical Use                    |
| ------------------ | -------------------------------------- | ------------------------------ |
| REQUIRED (default) | Join if exists, else start new         | Normal service calls           |
| REQUIRES_NEW       | Suspend current, start new             | Auditing/logging               |
| NESTED             | Create savepoint (rollback inner only) | Partial rollback inside parent |
| SUPPORTS           | Run within tx if one exists            | Read-only operations           |
| NOT_SUPPORTED      | Run non-transactionally                | Reporting                      |
| NEVER              | Throw error if a tx exists             | Safety guard                   |

### Example

```java
@Transactional
public void placeOrder() {
  saveOrder();           // same tx
  audit_requiresNew();   // separate tx
  throw new RuntimeException("outer fails");
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void audit_requiresNew() {
  auditRepo.save(new AuditLog("order saved"));
}
```

Audit commits, even though the outer one rolls back.

---

## 📘 4. Read-only transactions

```java
@Transactional(readOnly = true)
public List<Order> listRecentOrders(int limit) {
  return orderRepo.findTopNByOrderByCreatedAtDesc(limit);
}
```

Hints ORM and drivers that **no writes** will occur.
Improves performance (skips dirty checking, can use read-only DB replicas).

---

## 🧮 5. Isolation levels and timeout

**Spring-only** attributes:

```java
@Transactional(
  isolation = Isolation.REPEATABLE_READ,
  timeout = 10,
  readOnly = true
)
public List<User> queryActiveUsers() { ... }
```

* `Isolation.READ_COMMITTED` (default) → prevents dirty reads.
* `Isolation.REPEATABLE_READ` → consistent view of rows.
* `Isolation.SERIALIZABLE` → full locking, slowest but safest.
* `timeout` (seconds) → rollback if exceeded.

Jakarta/JTA `@Transactional` lacks these attributes — you configure them via the datasource.

---

## 🧰 6. Catching exceptions & marking rollback

If you catch an exception, the framework assumes everything’s fine — **it will commit** unless you mark rollback manually.

```java
import org.springframework.transaction.interceptor.TransactionAspectSupport;

@Transactional
public void catchButRollback() {
  try {
    doWork();
  } catch (Exception e) {
    TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
  }
}
```

Jakarta equivalent:

```java
import jakarta.transaction.TransactionSynchronizationRegistry;
import jakarta.annotation.Resource;

@Resource
TransactionSynchronizationRegistry tsr;

@jakarta.transaction.Transactional
public void catchButRollbackJta() {
  try {
    doWork();
  } catch (Exception e) {
    tsr.setRollbackOnly();
  }
}
```

---

## 🧱 7. Self-invocation pitfall (proxy mechanics)

```java
@Service
public class MyService {
  @Transactional
  public void outer() {
    inner(); // self-call bypasses proxy — no tx applied
  }

  @Transactional(propagation = Propagation.REQUIRES_NEW)
  public void inner() { ... }
}
```

Spring proxies intercept **external** calls only.
Solutions:

* Call via another bean (`otherBean.inner()`).
* Use interface-based proxy injection.
* Or enable AspectJ mode (`@EnableAspectJAutoProxy(exposeProxy = true)`).

---

## ⚖️ 8. Choosing between Spring & Jakarta

| Feature                              | Spring `@Transactional`     | Jakarta/JTA `@Transactional` |
| ------------------------------------ | --------------------------- | ---------------------------- |
| `isolation`, `timeout`, `readOnly`   | ✅ yes                       | ❌ no                         |
| `rollbackFor`, `noRollbackFor`       | ✅ yes                       | ❌ no                         |
| `propagation` (REQUIRES_NEW, NESTED) | ✅ yes                       | limited (TxType.*)           |
| Multiple transaction managers        | ✅ yes                       | ❌ global TM only             |
| XA / JTA compatibility               | ✅ via JtaTransactionManager | ✅ native                     |
| Simplicity / portability             | moderate                    | lightweight                  |

**Rule of thumb:**
Use **Spring** if you rely on JPA, multiple datasources, or fine-grained control.
Use **Jakarta** if you’re targeting EE containers or minimal portability.

---

## 🧭 9. Targeting specific transaction managers

```java
@Transactional(transactionManager = "ordersTxManager")
public void writeOrder() { ... }

@Transactional(transactionManager = "billingTxManager")
public void writeInvoice() { ... }
```

Used when your app has multiple datasources or distinct persistence contexts.

---

## 🔐 10. Common pitfalls checklist

☑ Rollback only happens if the framework *sees* the exception.
☑ Checked exceptions need `rollbackFor` or `rollbackOn`.
☑ Self-calls bypass proxy transactions.
☑ `readOnly=true` is a hint, not a guarantee.
☑ Use `REQUIRES_NEW` for independent commits (audits, logs).
☑ Catching exceptions? Set rollback manually.
☑ Pick one annotation style consistently.
☑ Log transaction boundaries when debugging (`DEBUG org.springframework.transaction`).

---

## 🧾 Summary Table

| Concept                       | Spring syntax                   | Jakarta syntax               | Notes                               |
| ----------------------------- | ------------------------------- | ---------------------------- | ----------------------------------- |
| Rollback on RuntimeException  | ✔️ (default)                    | ✔️ (default)                 | —                                   |
| Rollback on checked Exception | `rollbackFor=Exception.class`   | `rollbackOn=Exception.class` | must opt-in                         |
| Independent tx                | `Propagation.REQUIRES_NEW`      | `TxType.REQUIRES_NEW`        | inner commit survives outer failure |
| Read-only                     | `@Transactional(readOnly=true)` | —                            | optimization hint                   |
| Isolation/timeout             | attributes                      | container config             | Spring feature                      |
| Nested tx                     | `Propagation.NESTED`            | —                            | savepoint-based, JDBC only          |

---

## 🚀 TL;DR Mental Model

* A transaction begins at the first `@Transactional` boundary.
* If an exception *escapes* the boundary, rollback occurs (runtime-only by default).
* Nested calls usually share the same tx unless marked otherwise.
* Caught exceptions commit unless you flag rollback manually.
* Read-only methods are lighter, safer for queries.
* Always log resolved propagation chains when debugging weird commits.

---

**Outcome:** You now have a complete transactional playbook — clear rollback rules, predictable propagation, and a memory-safe mental map for every `@Transactional` boundary.
