---
title: Architecture Manifest and ADRs
date: 2026-11-09
tags:
  - architecture-manifest
  - adr
  - governance
  - contract
  - constraints
aliases:
  - Architecture Manifest
  - ADRs
  - Architecture Invariants
status: completed
---

# Architecture Manifest and ADRs

Once Phase 0 architecture discovery is completed, the agreed decisions must be formalized into machine-readable and human-readable records:

1. **`architecture_manifest.json`**: The authoritative machine-readable contract consumed by the Orchestrator, linters, and prompt hydrators.
2. **Architecture Decision Records (ADRs)**: Version-controlled Markdown documents explaining *why* decisions were made.

---

## The `architecture_manifest.json` Specification

Below is an enterprise example of an approved architectural manifest:

```json
{
  "$schema": "https://factory.internal/schemas/architecture-manifest-v1.json",
  "project_id": "enterprise-order-system",
  "version": "1.0.0",
  "architectural_style": "EVENT_DRIVEN_MICROSERVICES",
  
  "design_patterns": [
    "TRANSACTIONAL_OUTBOX",
    "SAGA_ORCHESTRATION",
    "HEXAGONAL_ARCHITECTURE",
    "DOMAIN_DRIVEN_DESIGN"
  ],

  "technology_matrix": {
    "backend": {
      "language": "Java 21",
      "framework": "Spring Boot 3.3.x",
      "build_system": "Gradle (Kotlin DSL)",
      "persistence": "PostgreSQL 16 via Spring Data JPA",
      "messaging": "Apache Kafka"
    },
    "frontend": {
      "framework": "Angular 18",
      "state_management": "NgRx",
      "styling": "TailwindCSS",
      "build_tool": "Vite"
    },
    "infrastructure": {
      "containerization": "Docker multi-stage builds",
      "orchestration": "Kubernetes Helm charts"
    }
  },

  "hard_invariants": [
    "NO direct database-to-database communication across services",
    "ALL state-changing REST endpoints must emit an event via the Outbox table in the same DB transaction",
    "FORBIDDEN: Python, Node.js backend services, and direct HTTP synchronous service-to-service calls for order workflows",
    "All DTOs and API contracts must be defined in OpenAPI 3.1 specifications"
  ]
}
```

---

## Enforcing Invariants in Worker Prompts

The Orchestrator injects the `hard_invariants` section directly into the system prompt of every Coder Agent:

```markdown
## STRICT ARCHITECTURAL INVARIANTS
You are working on an enterprise codebase governed by `architecture_manifest.json`.
The following rules are ABSOLUTE. Violating any of these rules will cause the linter to fail and reject your task:
- Language: Java 21 / Spring Boot 3 only.
- You are STRICTLY FORBIDDEN from generating Python or Node.js code.
- Every state mutation MUST write an event to the `outbox_events` table in the same database transaction.
- NO cross-service direct database queries.
```

---

## Architecture Decision Records (ADRs)

In addition to the JSON manifest, the factory generates human-readable ADRs stored in `docs/adr/`:

### Example: `ADR-001-transactional-outbox-pattern.md`
```markdown
# ADR 001: Adoption of Transactional Outbox Pattern for Inter-Service Events

## Status
Accepted

## Context
When an Order is created, the Inventory Service and Payment Service must be notified. 
Writing to the database and sending an event to Kafka in two separate steps introduces dual-write failures (e.g., database succeeds, but network partition prevents Kafka publish).

## Decision
All services will implement the Transactional Outbox Pattern. Events will be written to an `outbox_events` table within the same ACID transaction as the domain entity. A dedicated Debezium CDC (Change Data Capture) or background polling worker will publish events from the Outbox table to Apache Kafka.

## Consequences
- Positive: Guarantees zero lost events; guarantees at-least-once delivery.
- Negative: Requires consumers to be idempotent to handle potential duplicate messages.
```

---

## Automated Architectural Linter

The factory does not rely solely on LLMs to obey invariants. It runs automated architectural tests (such as **ArchUnit** in Java or custom AST linters in CI):

```java
// ArchUnit test running in the verification sandbox
@ArchTest
public static final ArchRule no_direct_cross_service_db_calls = 
    noClasses().that().resideInAPackage("..order..")
    .should().dependOnClassesThat().resideInAPackage("..payment.repository..");
```

If an agent writes code that violates an architectural boundary, ArchUnit fails the build, and the compiler traceback is fed back to the agent for self-repair.

---

## Related Notes
- [[01-guided-architecture-discovery|Guided Architecture Discovery (Phase 0)]]
- [[03-factory-archetypes-and-chassis|Factory Archetypes and Chassis Platforms]]
- [[04-evolutionary-architecture|Evolutionary Architecture: Monolith to Microservices]]
- [[01-context-scoping-and-contract-first|Context Scoping and Contract-First Development]]
