---
title: Agent Registry and Blueprints
date: 2026-11-09
tags:
  - agent-registry
  - blueprints
  - yaml-spec
  - tools
  - permissions
aliases:
  - Agent Registry
  - Worker Blueprints
status: completed
---

# Agent Registry and Blueprints

To prevent hardcoded prompts and arbitrary agent behaviors, a software factory uses an **Agent Registry**. 

The Agent Registry is a structured directory of version-controlled blueprints that define the persona, model, allowed tools, resource limits, and execution templates for each specialized worker role.

---

## Registry Directory Structure

```
factory-core/
└── agents/
    ├── registry/
    │   ├── architect_planner.yaml
    │   ├── backend_coder.yaml
    │   ├── frontend_coder.yaml
    │   ├── test_runner.yaml
    │   ├── code_reviewer.yaml
    │   └── fixer_integrator.yaml
    └── templates/
        ├── architect.jinja2
        ├── backend_coder.jinja2
        └── reviewer.jinja2
```

---

## Concrete Blueprint Specifications

### 1. Backend Coder Blueprint (`agents/registry/backend_coder.yaml`)
```yaml
name: "BackendCoder"
version: "1.2.0"
description: "Implements server-side routes, database queries, and business logic."

# Model configuration
model:
  provider: "anthropic"          # or "google", "openai"
  name: "claude-3-5-sonnet"
  temperature: 0.1               # Low temperature for deterministic syntax
  max_tokens: 8192

# Prompt template binding
prompt_template: "agents/templates/backend_coder.jinja2"

# Allowed sandbox tools
tools:
  - name: "read_file"
    description: "Read contents of files inside the sandbox"
  - name: "edit_file"
    description: "Make targeted line-based edits to files"
  - name: "run_bash"
    description: "Execute non-interactive terminal commands"
  - name: "query_symbol"
    description: "Lookup types and signatures from the Central AST"

# Operational boundaries & circuit breakers
guardrails:
  max_iterations: 15             # Maximum agent tool-calling turns
  timeout_seconds: 600           # 10 minutes maximum runtime
  disallowed_commands:
    - "rm -rf"
    - "curl"
    - "ssh"
    - "git push"                 # Workers cannot push directly to remote
```

### 2. Architect / Planner Blueprint (`agents/registry/architect_planner.yaml`)
```yaml
name: "ArchitectPlanner"
version: "2.0.0"
description: "Decomposes PRDs into a validated Directed Acyclic Graph (DAG)."

model:
  provider: "google"
  name: "gemini-1.5-pro"         # Large context window for large specs
  temperature: 0.2

prompt_template: "agents/templates/architect.jinja2"

tools:
  - name: "read_repo_tree"
  - name: "read_architecture_manifest"
  - name: "validate_dag_schema"

guardrails:
  max_iterations: 5
  timeout_seconds: 180
  output_format: "JSON_STRICT"   # Enforced JSON Schema
```

---

## Tool Permission Sandboxing

Not every agent gets the same tools:
- **Architect Agents**: Have read-only access to repository structures and the `architecture_manifest.json`. They have **no bash execution tools** and cannot edit source code.
- **Coder Agents**: Have file write tools and bash tools, but their writes are strictly restricted to their assigned `file_system_scope`.
- **Reviewer Agents**: Have read-only tools and a `post_pr_review` tool. They cannot alter files.

This principle of least privilege ensures that an agent cannot accidentally execute harmful commands or cross task boundaries.

---

## Related Notes
- [[01-ephemeral-worker-lifecycle|Ephemeral Worker Lifecycle and Elasticity]]
- [[03-execution-sandboxes-and-isolation|Execution Sandboxes and Isolation]]
- [[02-just-in-time-prompt-hydration|Just-In-Time (JIT) Prompt Hydration]]
- [[01-dag-specification-and-generation|DAG Specification and Generation]]
