---
name: project-orchestration
description: >
  This skill should be used when the user asks to "initialize a new project from a PRD", "scaffold a full-stack project", "create backend and frontend repos", "set up a new project", "generate CLAUDE.md from requirements", "start a project from scratch", or "orchestrate project creation".
---

# Project Orchestration

## Overview

Initializes a complete full-stack project from a PRD. Produces the root `CLAUDE.md` (permanent agent entry point), creates the necessary repositories, and populates the Module/Feature map that governs all future work.

<HARD-GATE>
**CONNECTIVITY REQUIRED.** Do NOT create any repository without active internet access. Composer, npm, and Docker image pulls will fail silently or partially. Verify connectivity first. If unavailable, generate only the root CLAUDE.md and stop — inform the user to re-run once the network is available.
</HARD-GATE>

## Anti-Pattern: "Scaffold Now, Document Later"

Generating repo scaffolding before producing the root `CLAUDE.md` and module/feature inventory. This leaves the project without a source of truth, forces retroactive reconciliation, and results in a map that never fully reflects reality. **Documentation and structure are generated in lockstep.**

## Checklist

1. **Verify connectivity** — abort repo creation if offline
2. **Read the PRD in full** — do not skim
3. **Decide repo scope** — Backend only / Frontend only / Both
4. **Run PRD Analysis** (see `prd-analysis` skill) — produces PRD-SUMMARY and module/feature inventory
5. **Generate root `CLAUDE.md`** — fill every section from PRD data; mark unknowns as `[PENDIENTE: reason]`
6. **Create Backend repo** — run ARCHITECTURE.md Phase 0 + Phase 1 if backend is needed
7. **Create Frontend repo** — run FRONTEND-ARCHITECTURE.md Phase 0 + Phase 1 if frontend is needed
8. **Verify each repo** — build, migrations/tests, Sail/Docker must pass cleanly
9. **Populate Module/Feature Map** — all modules and features listed with status `Pendiente`
10. **Delete this orchestration file** — root `CLAUDE.md` is now the entry point

## Process Flow

```dot
digraph orchestration {
    rankdir=TB;
    node [shape=box, style=rounded];

    A [label="Receive PRD"];
    B [label="Check Connectivity"];
    C [label="PRD Analysis\n(prd-analysis skill)"];
    D [label="Generate root CLAUDE.md"];
    E {shape=diamond, label="Backend\nneeded?"};
    F [label="Create Backend Repo\n(ARCHITECTURE.md Ph0+Ph1)"];
    G {shape=diamond, label="Frontend\nneeded?"};
    H [label="Create Frontend Repo\n(FRONTEND-ARCHITECTURE.md Ph0+Ph1)"];
    I [label="Verify Both Repos"];
    J [label="Populate Module/Feature Map"];
    K [label="Delete ORQUESTATOR.md"];
    STOP [label="STOP — No Network\nGenerate CLAUDE.md only", shape=ellipse];

    A -> B;
    B -> STOP [label="offline"];
    B -> C [label="online"];
    C -> D;
    D -> E;
    E -> F [label="yes"];
    E -> G [label="no"];
    F -> G;
    G -> H [label="yes"];
    G -> I [label="no"];
    H -> I;
    I -> J;
    J -> K;
}
```

## The Process

### Step 1 — Connectivity Check
Run a simple network probe before any `curl | bash` or `npx` commands. If offline, generate only the root `CLAUDE.md` with all repo sections marked `[PENDIENTE: repo not created — no connectivity]` and stop.

### Step 2 — PRD Analysis
Delegate to the `prd-analysis` skill. The output is:
- Structured list of backend **modules** with complexity levels and dependencies
- Structured list of frontend **features** with complexity levels and dependencies
- External integrations inventory
- Core business rules and key flows

### Step 3 — Root CLAUDE.md Generation
Fill the template below using PRD Analysis output. Use real names from the PRD for repos. If names are not specified, use `[nombre-repo-backend]` / `[nombre-repo-frontend]` as placeholders and ask the user.

**Required sections:**
- Project description (2-3 lines)
- Repo table (only repos that will exist)
- Tech stack (backend/frontend/contract — omit what doesn't apply)
- Business objective, actors, external integrations
- Module/Feature Map (all entries with status `Pendiente`)
- Documentation index (where to find what)
- Work protocol (before / during / after each task)
- Breaking change protocol
- Key business rules (extracted verbatim from PRD)
- Main flows (2-3 lines each)
- Implementation priority (phases with justification)

**Rules for the map:**
- Never invent endpoints — if not specified in PRD, use RESTful pattern and mark `[propuesto]`
- Default complexity to Nivel 2 if unclear, mark `[por confirmar]`
- Every business rule and invariant from the PRD must appear in "Reglas de Negocio Clave"

### Step 4 — Backend Repo Creation
If backend is needed and the folder does not yet exist:
1. Execute ARCHITECTURE.md **Phase 0** (PRD analysis, module docs, CLAUDE.md content)
2. Execute ARCHITECTURE.md **Phase 1** (Laravel + Sail creation, modular structure, Auth migration, docs, initial commit)
3. Verify: `sail up` clean, `sail artisan migrate` passes, `sail test` green

If the folder already exists: verify it has `ARCHITECTURE.md`, root `CLAUDE.md`, `docs/PRD-SUMMARY.md`, and a `CLAUDE.md` per module.

### Step 5 — Frontend Repo Creation
If frontend is needed and the folder does not yet exist:
1. Execute FRONTEND-ARCHITECTURE.md **Phase 0**
2. Execute FRONTEND-ARCHITECTURE.md **Phase 1** (Next.js + shadcn/ui + Vitest + Storybook + Docker)
3. Verify: `npm run dev` clean, `npm run build` passes, `npm run test` green, `docker compose build` succeeds

### Step 6 — Final Verification
- Module/Feature Map reflects all created modules and features
- Every repo has its own CLAUDE.md, PRD-SUMMARY.md, and per-module/feature CLAUDE.md
- Both repos compile and pass tests independently
- Root CLAUDE.md "Dónde Encontrar Documentación" table is accurate

## After Completion

**Documentation (ALWAYS update before closing the task):**
- Root `CLAUDE.md` — Module/Feature Map column "Estado" should read `Pendiente` for all entries
- Confirm repo table lists only repos that actually exist
- Delete `ORQUESTATOR.md` — it is now superseded
- First ADR in each repo (`001-modular-monolith.md` / `001-feature-based-architecture.md`) must be committed

## Key Principles

- **Root CLAUDE.md is the permanent entry point** — every subsequent agent session starts here
- **No repo without documentation** — CLAUDE.md per module/feature is non-negotiable
- **Offline = no scaffolding** — partial initialization is worse than none
- **Map = contract** — the Module/Feature Map is the single source of truth for the backend↔frontend contract; it must stay accurate at all times
- **One commit includes everything** — code, structure, and documentation ship together
