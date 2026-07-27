# aura-decisions

Decisions and specs that apply across Aura repos, in one place.

- `decisions/` — ADRs: what we decided, why, and what we considered and rejected.
- `specs/` — cross-repo specs. Free-form markdown for now; convert later if needed.
- `aura-historical-context.md` - index of previous architectural and spec documents related to Aura, providing public links and synopses for better context.

## How it works

A decision is real when a human states it (here, or in an issue/PR that lands here).
Agents draft; humans decide. Anything can be amended by a later ADR — nothing is precious.

Format: [MADR-lite](decisions/template.md). Number sequentially, never reuse numbers.
Superseded ADRs stay in place, marked `superseded by ADR-NNNN`.
