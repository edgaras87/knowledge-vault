---
title: Content
date: 2025-11-15
tags: 
  - java
  - core 
  
summary: records, enums, generics, sealed types, pattern matching, immutability, interfaces, and more.
aliases:
  - Java Core Language — Index

---

# Java Core Language — Index

Core Java language features live here: syntax, type system, and constructs that the compiler/JLS define directly  
(not libraries, not Spring, not “enterprise glue”).

Use this area as your hub when you want to recall **how the language itself works**, not how a framework uses it.

---

## Content

| Name                                            | Focus / Topic                                           | Notes                              |
|------------------------------------------------|---------------------------------------------------------|------------------------------------|
| [Records](../records/index.md)                    | Immutable data carriers, value semantics, DTO/value types | Use for small data-holders, config, responses |
| [Enums](enums/index.md)                        | Enumerated types, flags, patterns, enum-specific tricks | Domain vocabularies, status codes |
| [Generics](generics/index.md)                  | Type parameters, wildcards, bounds, erasure             | Collections, APIs, type-safety    |
| [Sealed Types](sealed-types/index.md)          | Sealed classes/interfaces, restricted hierarchies       | Algebraic-style type modeling     |
| [Classes vs Records](classes-vs-records.md)    | When to pick class vs record, trade-offs                | Decision guide, refactoring notes |
| [Pattern Matching](pattern-matching/index.md)  | `instanceof` patterns, `switch` patterns, record patterns | Pairs well with sealed + records  |
| [Immutability](immutability/index.md)          | Immutable design, defensive copies, withers             | Value objects, thread-safety      |
| [Interfaces & Abstract Classes](interfaces-abstract/index.md) | Contracts, inheritance, default methods     | Polymorphism & API design         |

> You don’t need all of these files on day one.  
> Start with the ones you actively use (`records`, `enums`, `generics`), then stub the others as you grow into them.

---

## Suggested layout

For reference, this index assumes a directory layout like:

```text
docs/
└── cheatsheets/
    └── language/
        └── java/
            └── core/
                ├── index.md
                ├── records/
                │   └── index.md
                ├── enums/
                │   └── index.md
                ├── generics/
                │   └── index.md
                ├── sealed-types/
                │   └── index.md
                ├── pattern-matching/
                │   └── index.md
                ├── immutability/
                │   └── index.md
                ├── interfaces-abstract/
                │   └── index.md
                └── classes-vs-records.md
```
