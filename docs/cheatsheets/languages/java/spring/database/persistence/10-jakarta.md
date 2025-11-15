---
title: Jakarta 
date: 2025-11-15
tags: 
  - java
  - jakarta
  - jpa
  - persistence 
  
summary: A concise cheatsheet for Jakarta Persistence (JPA) annotations and concepts, focusing on entity mapping, relationships, and common patterns in Java applications.
aliases:
  - Jakarta Persistence Cheatsheet
---

# Jakarta Persistence — Index

**`jakarta.persistence.*` Annotation Map**

Jakarta Persistence (JPA) tells the runtime how Java classes, fields, and relationships turn into relational tables, columns, keys, and joins.
Think of it as the **schema + relationship DSL** for your entity domain.

This index lists every JPA annotation you should know, grouped by function, with short explanations and links to where you'd store deeper cheatsheets.

---

## 0) Contents

| Group                                             | Purpose                                        |
| ------------------------------------------------- | ---------------------------------------------- |
| [Entity-Level](#1-entity-level)                   | Mark a class as an entity + map to table       |
| [Primary Keys](#2-primary-key--id-generation)     | Identity & ID generation strategies            |
| [Columns](#3-column--field-mapping)               | Control SQL column shape                       |
| [Enumerations](#4-enum-mapping)                   | STRING vs ORDINAL                              |
| [Relationships](#5-relationships)                 | OneToOne, ManyToOne, etc.                      |
| [Join Controls](#6-join--foreign-key-annotations) | JoinColumn, JoinTable                          |
| [Embedded Types](#7-embedded-value-types)         | Value objects flattened into table             |
| [Lifecycle](#8-entity-lifecycle-callbacks)        | PrePersist, PostLoad, etc.                     |
| [Inheritance](#9-inheritance-strategies)          | Single table, joined, table per class          |
| [Other Mapping Tools](#10-other-mapping-tools)    | Converters, Lob, Transient                     |
| [Advanced Schema](#11-advanced-schema)            | SequenceGen, TableGen, Index, UniqueConstraint |

---

## 1) Entity-Level

### **`@Entity`**

Marks a class as a persistent entity managed by JPA.

### **`@Table`**

Configures table name, schema, indexes, unique constraints.

### **`@Access`**

Choose field access or property (getter) access.

### **`@Cacheable`**

Hints whether the entity is eligible for second-level caching (provider-specific).

---

## 2) Primary Key / ID Generation

### **`@Id`**

Marks the primary key field.

### **`@GeneratedValue`**

Configures ID generation strategy (`IDENTITY`, `SEQUENCE`, `TABLE`, `AUTO`).

### **`@SequenceGenerator`**

Defines a database sequence for ID generation (Postgres, Oracle style).

### **`@TableGenerator`**

Uses a special table for ID generation (rare; legacy).

---

## 3) Column / Field Mapping

### **`@Column`**

Controls column name, nullability, uniqueness, length, precision/scale.

### **`@Lob`**

Marks field as CLOB/BLOB.

### **`@Temporal`** *(legacy pre-Java 8)*

Specifies date/time type for `Date`.

### **`@Basic`**

Defines fetch type and optionality for simple fields (rarely overridden).

### **`@Transient`**

Field is ignored by JPA (not persisted).

### **`@Convert`**

Apply a custom AttributeConverter.

---

## 4) Enum Mapping

### **`@Enumerated`**

Choose enum storage: `STRING` (safe) or `ORDINAL` (dangerous).

---

## 5) Relationships

### **`@ManyToOne`**

Child → parent relation (FK lives here). Most common relationship.

### **`@OneToMany`**

Parent → collection of children (inverse of ManyToOne).

### **`@OneToOne`**

Exactly one entity bound to exactly one other.

### **`@ManyToMany`**

Many ↔ many via join table.

### **`@MapsId`**

Share the parent’s primary key (special 1:1 pattern).

### **`@OrderBy`**

Define ordering of a collection when fetched.

### **`@OrderColumn`**

Persistent order column for indexed lists.

---

## 6) Join / Foreign-Key Annotations

### **`@JoinColumn`**

Configures the FK column: name, nullability, referenced column.

### **`@JoinColumns`**

Composite foreign keys.

### **`@JoinTable`**

Defines join table for ManyToMany or OneToMany-with-join-table.

### **`@ForeignKey`**

Names an FK constraint explicitly (vendor-dependent).

---

## 7) Embedded Value Types

### **`@Embeddable`**

Marks class as a value object that doesn’t have its own table.

### **`@Embedded`**

Embed an `@Embeddable` field inside an entity; fields flatten into same table.

### **`@AttributeOverride` / `@AttributeOverrides`**

Override column mapping for fields inside an embeddable.

---

## 8) Entity Lifecycle Callbacks

### **`@PrePersist`**

Before insert.

### **`@PostPersist`**

After insert.

### **`@PreUpdate`**

Before update.

### **`@PostUpdate`**

After update.

### **`@PreRemove`**

Before delete.

### **`@PostRemove`**

After delete.

### **`@PostLoad`**

After entity loaded.

### **`@EntityListeners`**

External class to handle lifecycle callbacks.

---

## 9) Inheritance Strategies

### **`@Inheritance`**

Choose strategy: `SINGLE_TABLE`, `JOINED`, `TABLE_PER_CLASS`.

### **`@DiscriminatorColumn`**

Column specifying subtype.

### **`@DiscriminatorValue`**

Value identifying subtype.

---

## 10) Other Mapping Tools

### **`@NamedQuery` / `@NamedQueries`**

Predefined JPQL queries.

### **`@NamedNativeQuery`**

Predefined SQL queries.

### **`@SqlResultSetMapping`**

Map raw SQL to objects (advanced).

### **`@Converter`**

Class-level annotation for attribute converters.

---

## 11) Advanced Schema Controls

### **`@Index`** (inside `@Table`)

Defines DB index; `unique = true` makes a unique constraint.

### **`@UniqueConstraint`**

Old way to define uniqueness; often replaced by `@Index(unique = true)`.

### **`@SecondaryTable` / `@SecondaryTables`**

Map entity columns across multiple tables.

---

## 12) How to Think About All of This

If you need to map:

* **Class → Table** → `@Entity`, `@Table`
* **Field → Column** → `@Column`, `@Basic`, `@Lob`
* **Enum → String** → `@Enumerated`
* **FK → Table relationship** → `@ManyToOne`, `@JoinColumn`
* **Join table** → `@JoinTable`
* **Flatten value object** → `@Embeddable`, `@Embedded`
* **Lifecycle hooks** → `@PrePersist`, etc.
* **ID generation** → `@GeneratedValue`, `@SequenceGenerator`
* **Inheritance** → `@Inheritance`, `@DiscriminatorColumn`

Everything else is just details.
