# ADR-0002: Cross-repo decisions and specs live in aura-decisions

- Status: accepted
- Date: 2026-07-21
- Deciders: Adam (proposed in #agent-workroom), Philip. Ali may amend.

## Context

Decisions and specs spanning multiple Aura repos had no single home.
aura-workroom Issue #1 asked where the spec should canonically live.

## Decision

This repo is the home for cross-repo ADRs and specs. Markdown. ADRs follow
[MADR-lite](template.md); specs are free-form for now, converted later if needed.

## Rationale

Decisions outlive any single repo. One findable place beats per-repo scatter —
for humans and for agents loading context.

## Alternatives considered

- **`decisions/` directory inside aura-workroom** — couples long-lived decisions
  to one repo about one topic (agent collaboration).
- **Per-repo ADR directories** — right for repo-local decisions (still fine to
  use); wrong for cross-cutting ones.

## Consequences

Repo-local decisions stay in their repos; anything cross-cutting lands here.
Partially answers workroom Issue #1 — Ali can amend.
