---
name: bulletproof
description: Apply and enforce the Bulletproof domain-driven architecture pattern — six-layer domain structure (entities, storage, gateway, useCase, transport, setup) with SOLID boundaries. Use when designing new domains, reviewing layering violations, or refactoring toward clean domain separation.
---

# Bulletproof Domain-Driven Architecture Pattern

The bulletproof pattern structures backend services around clear domain boundaries with enforced SOLID principles. Here's what you need to know:

## Core Structure

Each domain contains six layers:
- **entities**: Pure domain types with invariants
- **storage**: Persistence interface + adapters (MySQL, PostgreSQL, MongoDB)
- **gateway**: Third-party integration interface + adapters
- **useCase**: Business logic orchestration
- **transport**: Inbound handlers (REST, gRPC)
- **setup**: Dependency injection wiring

## Key Rules

**Hard boundaries:**
1. Cross-domain imports allowed only from `entities` and `useCase` of other domains
2. DTOs translate between layers but never cross boundaries
3. Business logic stays in useCase—never in storage, gateway, or transport
4. One domain = one purpose; nest subdomains only when they have independent lifecycle

**Dependency flow:**
"Transport → useCase → storage interface ← storage/mysql" with gateway following the same pattern. Interfaces live in the domain; adapters implement them.

## SOLID Enforcement

The layout structurally prevents violations:

- **SRP**: Each layer has one reason to change (persistence engine, protocol, business rules, etc.)
- **OCP**: "Add a new adapter, never modify existing ones"
- **LSP**: Every adapter must be a true substitute for its interface
- **ISP**: Keep interfaces small—useCase depends only on methods it calls
- **DIP**: High-level depends on abstractions; low-level implements them

## Anti-patterns to Avoid

Flag violations like useCase importing concrete adapters, DTOs leaking across layers, business logic in storage/gateway/transport, or setup constructing dependencies based on runtime conditions.

This pattern makes layering violations structurally obvious during code review.
