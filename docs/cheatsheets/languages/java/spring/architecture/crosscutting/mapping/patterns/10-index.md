---
title: Content 
date: 2025-11-18
---



# DTO Shaping Patterns — Short Index

This folder contains focused mapping patterns you will reuse across your whole project.  
Each pattern solves a very specific transformation problem at the web edge.

---

## Patterns

- **WebMappingContext** — Shows how to pass ApiLinks, ProblemLinks, Locale, and Clock into mappers so they can produce correct web-facing DTOs.

- **Immutable Record Extra Fields** — Explains how to populate extra fields (links, URIs, policy values) in immutable record DTOs without mutating them.

- **Link Mapping Patterns** — Demonstrates how to generate self/collection/related links cleanly using helpers, qualifiers, and @Context.

- **Nested → Flat Mapping** — Shows how to flatten nested domain models into simple flat DTOs using MapStruct’s path-based mapping.

- **PATCH Update Mapping** — Provides a safe pattern for partial updates, applying only non-null fields onto existing domain or JPA objects.

---

More patterns can be added here as your project grows.


