---
title: Dynamic DAG Mutation and Blast Radius Analysis
date: 2026-11-09
tags:
  - dag
  - dynamic-dag
  - blast-radius
  - change-management
  - refactoring
aliases:
  - Dynamic DAG Mutation
  - Blast Radius
  - Subgraph Splicing
status: completed
---

# Dynamic DAG Mutation and Blast Radius Analysis

In enterprise application development, **requirements change mid-flight**. A client pivots, a compliance mandate alters data storage rules, or a frontend requirement reveals that an upstream database schema is inadequate.

A naive factory would have to scrap everything and restart from scratch. An advanced software factory treats the DAG as **dynamic and mutable**, using **Blast Radius Analysis** and **Subgraph Splicing**.

---

## 1. The Blast Radius: Invalidation Cascades

When a requirement changes, the Orchestrator does not wipe the repository. It calculates the **Blast Radius**—the exact subtree in the DAG affected by the change:

```mermaid
flowchart TD
    classDef untouched fill:#d4edda,stroke:#28a745,stroke-width:2px;
    classDef changed fill:#fff3cd,stroke:#ffc107,stroke-width:3px;
    classDef stale fill:#f8d7da,stroke:#dc3545,stroke-width:2px;
    classDef cancelled fill:#e2e3e5,stroke:#6c757d,stroke-width:2px;

    T1["Task 1: Auth & User DB"]:::untouched
    T2["Task 2: Billing DB Schema\n(Requirement Changed Here)"]:::changed
    T3["Task 3: User Profile UI"]:::untouched
    T4["Task 4: Stripe Webhook\n(Already Done -> STALE)"]:::stale
    T5["Task 5: Checkout UI\n(In-Flight -> CANCELLED)"]:::cancelled

    T1 --> T3
    T1 --> T2
    T2 -->|Invalidation cascade| T4
    T2 -->|Invalidation cascade| T5
```

### The State Classifications:
1. **Untouched / Immune (T1, T3)**: Nodes whose inputs and contracts remain unaffected. Their code and tests are preserved intact.
2. **Changed / Root Cause (T2)**: The earliest node where the new requirement deviates from the old specification.
3. **Stale / Dirty (T4)**: Nodes that already ran to completion, but were built on assumptions or types that are no longer valid.
4. **Cancelled (T5)**: In-flight workers currently executing. The Orchestrator immediately fires a cancellation signal to kill their sandbox containers and stop burning compute/tokens.

---

## 2. Bidirectional Ripple: When Downstream Breaks Upstream

Requirement changes rarely flow only downwards. Frequently, a change in a downstream requirement breaks an upstream foundation:

> **Example:** While implementing Task 4 (*Checkout UI*), the user demands: *"We need customers to pay with crypto (USDC/Ethereum), not just credit cards."*
> Although requested at the UI stage, this requires adding `wallet_address` and `transaction_hash` to the database schema (Task 2: Upstream).

### Subgraph Splicing Algorithm

```mermaid
flowchart TD
    classDef untouched fill:#d4edda,stroke:#28a745,stroke-width:2px;
    classDef replaced fill:#f8d7da,stroke:#dc3545,stroke-width:2px;
    classDef patch fill:#cce5ff,stroke:#004085,stroke-width:2px;

    T1["Task 1: User Auth"]:::untouched
    T2["Task 2: Billing DB (Stripe Only)"]:::replaced
    T3["Task 3: Stripe Client Service"]:::replaced
    T4["Task 4: Checkout UI"]:::replaced

    subgraph SubgraphReplacement["Re-DAGging: Subgraph Spliced"]
        P1["Patch 2: Migrate DB Schema\nAdd WalletAddress & CryptoTx"]:::patch
        P2["Patch 3: Implement Web3 Provider & Stripe"]:::patch
        P3["Patch 4: Checkout UI with WalletConnect"]:::patch
        
        P1 --> P2
        P2 --> P3
    end

    T1 --> P1
```

1. **PRD Diffing**: An agent compares `PRD_v1` vs. `PRD_v2`.
2. **Reverse Traceback**: The Planner traces upward to locate the earliest ancestor affected (`Task 2`).
3. **Invalidate Subtree**: The Orchestrator invalidates `Task 2` and its descendants.
4. **Splice Patch DAG**: The Planner outputs a replacement sub-DAG (`Patch 2`, `Patch 3`, `Patch 4`) and inserts it into the graph.

---

## 3. Git Atomic Checkpoints and Rollbacks

Every completed task in the factory automatically commits its result with a deterministic Git tag:
`checkpoint/<task_id>` (e.g., `checkpoint/T1_auth`).

When a subtree is invalidated:
- The Orchestrator performs a `git checkout` to the last clean, unaffected ancestor commit (`checkpoint/T1_auth`).
- Unaffected branches continue their work without interruption.

---

## 4. Rewrite vs. Refactor Decision Matrix

When tasks are invalidated, the factory decides between two recovery modes:

| Mode | When Used | Mechanism |
| :--- | :--- | :--- |
| **Clean Invalidation (Scrap & Redo)** | Early stage, or small isolated tasks (< 200 lines). | Discard branch diffs. Re-run clean task from new spec. |
| **Migration / Refactor Mode** | Late stage, complex enterprise codebases with existing tests. | Spawn a **Refactor Agent** with the existing code + new spec + compiler errors. |

---

## 5. Compiler-as-a-Sensor for Guided Refactoring

In enterprise systems, do not ask an LLM to guess what broke after a requirement change. Use the **deterministic compiler/type-checker as a sensor**:

```mermaid
flowchart LR
    ChangeSpec[Update Core Types / Schema] --> RunCompiler[Run 'tsc --noEmit' or 'mvn compile']
    RunCompiler --> ErrorList[Collect Deterministic Compiler Errors]
    ErrorList --> Partition[Partition Errors by File Domain]
    Partition --> Workers[Dispatch Targeted Fixer Agents]
    Workers --> GreenBuild[All Compiler Errors Resolved & Tests Pass]
```

1. Update the database schema and type definitions.
2. Run the compiler (`tsc --noEmit` or `mvn compile`).
3. The compiler emits an exact list of every broken file and method signature across the repository.
4. The Orchestrator groups these errors by file domain and dispatches parallel **Fixer Agents** to resolve them.

---

## Related Notes
- [[01-dag-specification-and-generation|DAG Specification and Generation]]
- [[02-event-driven-dependency-resolution|Event-Driven Dependency Resolution]]
- [[02-architecture-manifest-and-adrs|Architecture Manifest and ADRs]]
- [[04-evolutionary-architecture|Evolutionary Architecture: Monolith to Microservices]]
