---
name: prd-analysis
description: >
  This skill should be used when the user asks to "analyze a PRD", "identify modules from requirements", "extract features from a spec", "assign complexity levels", "decompose a product requirements document", "identify business rules from PRD", or "plan architecture from a document".
---

# PRD Analysis

## Overview

Transforms a raw PRD into structured architectural inputs. The output feeds both the root `CLAUDE.md` (via `project-orchestration` skill) and the per-repo documentation (`ARCHITECTURE.md` Phase 0 / `FRONTEND-ARCHITECTURE.md` Phase 0). This skill is reusable independently — it can be run on a PRD without triggering full project scaffolding.

<HARD-GATE>
**Never invent what the PRD doesn't say.** If a module, feature, endpoint, rule, or integration is not mentioned or clearly inferable from the PRD, mark it `[PENDIENTE: not specified in PRD]`. Do not fill gaps with assumptions.
</HARD-GATE>

## Anti-Pattern: "Skeleton First"

Jumping directly to complexity levels and module names without reading the full PRD first. This results in missing business rules buried in later sections, incorrect dependency graphs, and modules that don't map to real product needs.

## Checklist

1. **Read the entire PRD** before writing anything
2. **Extract actors/users** — who uses the system and how
3. **Extract business rules** — every invariant, constraint, and validation
4. **Identify external integrations** — third-party APIs, payment gateways, auth providers
5. **Identify backend modules** — group by business responsibility, not by technical layer
6. **Identify frontend features** — group by user-facing capability
7. **Assign complexity levels** — justify each level with evidence from the PRD
8. **Build dependency graph** — which module/feature depends on which
9. **Identify events** — which modules emit domain events and who listens
10. **Define implementation priority** — based on technical dependencies, not business preference alone
11. **Generate `docs/PRD-SUMMARY.md`**

## Process Flow

```dot
digraph prd_analysis {
    rankdir=LR;
    node [shape=box, style=rounded];

    A [label="Read full PRD"];
    B [label="Extract:\nActors\nBusiness Rules\nIntegrations"];
    C [label="Identify\nBackend Modules"];
    D [label="Identify\nFrontend Features"];
    E [label="Assign Complexity\nLevels (1/2/3)"];
    F [label="Build Dependency\nGraph"];
    G [label="Identify Events\n(emit / listen)"];
    H [label="Define Priority\nPhases"];
    I [label="Generate\nPRD-SUMMARY.md"];

    A -> B -> C -> E -> F -> G -> H -> I;
    B -> D -> E;
}
```

## The Process

### Step 1 — Actors and Context
List every user type, system actor, and role. For each, describe: what they can do, what data they own, and what their primary workflows are.

### Step 2 — Business Rules Extraction
Scan the entire PRD for:
- Invariants ("a user can only have one active subscription")
- Validations ("email must be unique")
- Restrictions ("orders cannot be cancelled after 24 hours")
- Calculations ("final price = base + tax − discount")

Each rule goes into the root `CLAUDE.md` under "Reglas de Negocio Clave" and into the relevant module/feature `CLAUDE.md`.

### Step 3 — Module Identification (Backend)
Group functionality by **business responsibility**, not by technical layer. Each module owns:
- Its data (tables/models)
- Its logic (services)
- Its API surface (controllers + routes)

**Complexity level rules:**
- **Nivel 1** — Pure CRUD, no business logic beyond validation, no events. Example: tags, categories, config tables.
- **Nivel 2** — Standard module with services, events, domain rules. Example: orders, profiles, notifications.
- **Nivel 3** — External integrations, complex workflows, multi-step processes, heavy calculations. Example: payments, auth with OAuth, real-time features.

If not enough information exists to determine level, default to **Nivel 2** and mark `[por confirmar]`.

### Step 4 — Feature Identification (Frontend)
Group by **user-facing capability**, mapping to sections of the UI or distinct user workflows.

**Complexity level rules:**
- **Nivel 1** — Presentation only, no data fetching, no state. Example: landing page, static content.
- **Nivel 2** — Data fetching, forms, local state, Server + Client components. Example: product listing, profile editor.
- **Nivel 3** — Complex state machines, multi-step flows, third-party embeds, real-time UI. Example: checkout wizard, live dashboard, rich text editor.

### Step 5 — Dependency Graph
For each module/feature:
- What does it **depend on** (consumes via ModuleApi or feature export)?
- What **depends on it** (is consumed by other modules/features)?
- What **events does it emit**?
- What **events does it listen to**?

Identify circular dependencies and flag them — they indicate a wrong boundary in the module/feature split.

### Step 6 — Implementation Priority
Order phases by **technical dependency**, not by business priority:
1. Auth (everything else depends on it)
2. Core domain modules with no dependencies
3. Modules that depend on core
4. Secondary / peripheral features

### Step 7 — PRD-SUMMARY.md
Generate `docs/PRD-SUMMARY.md` with this structure:

```markdown
# [Project Name] — PRD Summary

## Description
[2-3 lines: what it is and what problem it solves]

## Business Objective
[Primary goal]

## Actors / Users
| Actor | Role | Primary Actions |
|-------|------|-----------------|

## Backend Modules
| Module | Responsibility | Level | Depends On | Emits Events | Listens To |
|--------|---------------|-------|-----------|-------------|-----------|

## Frontend Features
| Feature | Responsibility | Level | Depends On | Server/Client boundary |
|---------|---------------|-------|-----------|----------------------|

## External Integrations
| Service | Purpose |
|---------|---------|

## Key Business Rules
- [rule 1]
- [rule 2]

## Technical Requirements
- [requirements that impact architectural decisions]
```

## After Completion

**Documentation (ALWAYS update before closing the task):**
- `docs/PRD-SUMMARY.md` — must be saved to the repository before Phase 1 begins
- Root `CLAUDE.md` — "Reglas de Negocio Clave" populated from this analysis
- Module/Feature Map in root `CLAUDE.md` — pre-populated with all identified modules and features

## Key Principles

- **Read first, write second** — full PRD read is non-negotiable before any output
- **Justify every level** — complexity levels without PRD evidence are assumptions, not analysis
- **Business rules are invariants** — anything the system must never allow belongs in the rules section
- **Boundaries define independence** — a well-drawn module/feature boundary means no circular dependencies
- **PRD-SUMMARY is the stable reference** — once generated, it doesn't change unless the PRD changes
