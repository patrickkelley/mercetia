# AGENTS.md

## What this repo is

Docs-only specification repo for **Mercetia Core**, a planned Java 21 / Gradle payroll calculation engine. There is no application code, build config, CI, or tests — nothing to run or compile. The sole artifact is `docs/PRD.md` (v2.1, ~1400 lines, "Approved for Phase 1 build").

## Workflow conventions

- All changes are PRD edits, not code. Read `docs/PRD.md` fully before proposing any change; it is the single source of truth.
- Work happens on feature branches named `prd/<topic>` (e.g. `prd/mercetia-core-only`), merged into `master` via GitHub PRs. Current branch: `prd/mercetia-core-only`.
- Commit messages use short conventional prefixes: `docs:`, `refactor:`, `PRD:`.

## PRD structural conventions (preserve these)

- Every requirement carries a canonical ID: `REQ-*` marker plus `R<number>` (e.g. `REQ-HOURS-CAP / R11`). Requirements are referenced by graph node (P→G→F→R→M: Persona → Goal → Feature → Requirement → Metric). Do not introduce requirements without IDs, and do not renumber or merge existing R-numbers.
- Changes to requirements must stay traceable: update the Graph Audit Summary (§12.1 / top table), mermaid dependency graph (§0), and acceptance criteria (§14) when the graph changes.
- Out-of-scope items carry `scope=deferred` markers and a `REQ-OOS-*` ID (§9.4); Phase 1 scope is `mercetia-core` only (pure Java, zero Spring imports).
- ADRs live in §13; new architecture decisions append `ADR-0xx`.
- `{toc}` at the top is an unrendered placeholder — leave it.

## Known file quirks

- The file contains stale leftovers from the old "Mercetia Calculator" scope that were never fully cleaned when narrowed to `mercetia-core`: a duplicate `### 1. Project Overview` header (lines ~171–177) and a duplicate `### 3.1 Performance (SLA)` header (lines ~419–421). Do not "fix" these silently — flag them to the user before removing.
- Sections 5–8, 10–14 still describe API/CLI/security modules (out of Phase 1 scope); they are retained intentionally for later phases, not dead content.

## Verification

No automated checks exist. Verify edits by re-reading the affected sections and checking R-number / REQ-* / AUD-* references remain consistent (duplicate-defect history: AUD-01…AUD-43).