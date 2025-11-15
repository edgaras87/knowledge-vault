---
title: '@OneToOne' 
date: 2025-11-15
tags: 
  - database
  - jpa
  - hibernate
  - jakarta persistence 
  
summary: Comprehensive cheatsheet for the Jakarta Persistence `@OneToOne` annotation, detailing its purpose, parameters, and best practices for modeling one-to-one relationships between entities.
aliases:
  - Jakarta Persistence — @OneToOne Cheatsheet

---

# Jakarta Persistence — `@OneToOne` Cheatsheet

Package: `jakarta.persistence.OneToOne`

`@OneToOne` models a **one-to-one** relationship:

* One User → One Profile
* One Product → One Inventory record
* One Order → One Payment

But the database must enforce this, either through:

* a **unique foreign key**, or
* **shared primary keys**.

---

## 0) Contents

| Section                                  | Description                     | Link                                                         |
| ---------------------------------------- | ------------------------------- | ------------------------------------------------------------ |
| **1) All Parameters**                    | Complete @OneToOne signature    | → [All Parameters](#1-all-parameters)                        |
| **2) Mental Model**                      | Two ways to implement 1:1 in DB | → [Mental Model](#2-mental-model)                            |
| **3) FK-Based One-to-One (recommended)** | The “unique FK” style           | → [FK-Based One-to-One](#3-fk-based-one-to-one--recommended) |
| **4) Bidirectional One-to-One**          | Owning vs inverse               | → [Bidirectional](#4-bidirectional-one-to-one)               |
| **5) Shared Primary Key (advanced)**     | PK = FK                         | → [Shared PK](#5-shared-primary-key-one-to-one--advanced)    |
| **6) Cascade, Fetch, Orphans**           | Behavior details                | → [Cascade & Behavior](#6-cascade--fetch--orphanremoval)     |
| **7) Practical Use Cases**               | Real patterns                   | → [Use Cases](#7-practical-use-cases)                        |
| **8) Best Practices**                    | What to memorize                | → [Best Practices](#8-best-practices)                        |

---

## 1) All Parameters

```java
@OneToOne(
    mappedBy = "",
    cascade = {},
    fetch = FetchType.EAGER,   // default (bad)
    optional = true,
    orphanRemoval = false,
    targetEntity = void.class
)
```

Key points:

* **`mappedBy`** → inverse side
* **`optional`** → nullable FK
* **`fetch`** → default is **EAGER** (bad, override to LAZY)
* **`cascade`** → propagate operations
* **`orphanRemoval`** → delete children removed from reference

---

## 2) Mental Model

There are *two fundamentally different* physical shapes in the database:

---

### 2.1 **Unique FK** (most common)

```
table user_profiles
   user_id BIGINT NOT NULL UNIQUE
   FK → users.id
```

This means:

* profile belongs to exactly one user
* user has at most one profile
* FK column has **unique** constraint

JPA mapping uses:

```java
@JoinColumn(name = "user_id", unique = true)
```

---

### 2.2 **Shared Primary Key** (advanced)

```
user_profiles.id PK/FK → users.id
```

Profile's primary key = User's primary key.

Useful when:

* profile cannot exist without user
* tight 1:1 identity
* strict domain integrity

Requires `@MapsId`.

---

## 3) FK-Based One-to-One (recommended)

Owning side holds the FK.

### Example: User ↔ UserProfile

### Profile (owning side)

```java
@Entity
@Table(name = "user_profiles")
public class UserProfileEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @OneToOne(optional = false, fetch = FetchType.LAZY)
  @JoinColumn(
    name = "user_id",
    nullable = false,
    unique = true
  )
  private UserEntity user;

  @Column(length = 500)
  private String bio;
}
```

### User (inverse side)

```java
@Entity
@Table(name = "users")
public class UserEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @OneToOne(mappedBy = "user", fetch = FetchType.LAZY)
  private UserProfileEntity profile;

  private String email;
}
```

**`unique = true`** enforces the 1:1.

---

## 4) Bidirectional One-to-One

Own side:

```java
@OneToOne
@JoinColumn(name = "user_id", unique = true)
private UserEntity user;
```

Inverse side:

```java
@OneToOne(mappedBy = "user")
private UserProfileEntity profile;
```

**Owning side = the side with `@JoinColumn`**.
Inverse side uses only `mappedBy`.

Changes must be applied **on owning side**, e.g.:

```java
profile.setUser(user);
user.setProfile(profile);
```

---

## 5) Shared Primary Key One-to-One (advanced)

Ideal for “dependent identity” cases.

```
profiles.id = users.id
```

### User

```java
@Entity
public class UserEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;
}
```

### Profile

```java
@Entity
@Table(name = "user_profiles")
public class UserProfileEntity {

  @Id
  private Long id; // same as user.id

  @OneToOne(optional = false)
  @MapsId                      // critical
  @JoinColumn(name = "id")
  private UserEntity user;

  private String bio;
}
```

Effects:

* profile PK comes from parent
* profile cannot exist without user
* extremely strict integrity
* great for “profile is always exactly one extension of user”

This is more advanced but powerful.

---

## 6) Cascade, Fetch, OrphanRemoval

### Fetch Type

Default:

```java
@OneToOne(fetch = FetchType.EAGER)  // default: terrible
```

Always override:

```java
@OneToOne(fetch = FetchType.LAZY)
```

### Cascade

Safe:

* `CascadeType.PERSIST`
* `CascadeType.MERGE`

Caution:

* `CascadeType.REMOVE` → removes the child when parent removed
* `CascadeType.ALL` → apply all above
* Only use when child belongs exclusively to parent

### Orphan Removal

```java
@OneToOne(orphanRemoval = true)
private ProfileEntity profile;
```

If you set profile to null:

```java
user.setProfile(null);
```

→ deletes that profile row automatically.

Great for 1:1 “component-like” entities.
Dangerous if child is shared (it shouldn't be in a 1:1 anyway).

---

## 7) Practical Use Cases

---

### 7.1 User ↔ UserProfile

FK-based:

* user has profile
* profile belongs to user
* profile’s lifecycle often tied to user → cascade

### 7.2 Product ↔ Inventory

Shared PK:

* inventory PK = product PK
* inventory cannot exist alone
* good for extension models

### 7.3 Order ↔ Payment

Depends on domain:

* might be FK-based (if payment created after order)
* shared PK if payment always must exist

---

## 8) Best Practices

### 1. Prefer **unique-FK 1:1** over shared PK

Simplest. Most flexible. Clear in DB.

### 2. Always make 1:1 **LAZY**

EAGER is dangerous — prevents batching, tends to explode queries.

### 3. Cascade only when child belongs to parent

Otherwise you risk deleting important rows.

### 4. Use `orphanRemoval = true` **only** for truly dependent children

(Profiles, details records, settings tables)

### 5. The owning side = side with `@JoinColumn`

This is the side that writes the FK.

### 6. Avoid unidirectional 1:1 unless necessary

Bidirectional is clearer and avoids weird FK issues.

