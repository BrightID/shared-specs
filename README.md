# shared-specs

ADRs and specs shared across two or more BrightID repos, in one place.

- `decisions/` — ADRs: what we decided, why, and what we considered and rejected.
  See [How ADRs work](#how-adrs-work) below.
- `openspec/` — this repo is also an OpenSpec [store](https://github.com/Fission-AI/OpenSpec/blob/main/docs/stores-beta/user-guide.md): a standalone `openspec/` planning root (specs + changes) that other BrightID repos reference read-only, so a spec spanning more than one repo has exactly one home instead of being copy-pasted into each.
  It currently holds only the store scaffolding — no specs have been migrated in yet.

This repo does **not** hold the BrightID/Aura big-picture design doc — that's [BrightID/foundations](https://github.com/BrightID/foundations).
Reference this repo as an OpenSpec store (`references:`) for concrete, shareable specs.
Reference `foundations` as a compass in your own `context:` field instead — it's one evolving narrative document, not a collection of discrete specs, so OpenSpec's store mechanism doesn't fit it.

## How ADRs work

A decision is real when a human states it (here, or in an issue/PR that lands here).
Agents draft; humans decide.
An ADR is never edited after acceptance — only superseded by a later ADR that says so explicitly.
Nothing here is precious, but the record of what was decided and when stays intact.

Format: [MADR-lite](decisions/template.md).
Number sequentially, never reuse numbers.
Superseded ADRs stay in place, marked `superseded by ADR-NNNN`.

## One-time setup (each developer, each machine)

Registering a store — telling your local OpenSpec CLI where it lives on disk — is machine-local and never committed, so every developer runs it once.

1. **Install/upgrade the CLI**:

   ```bash
   npm install -g @fission-ai/openspec@latest
   openspec --version   # 1.6.0 or newer — that's when Stores shipped
   ```

2. **Clone this repo**:

   ```bash
   git clone https://github.com/BrightID/shared-specs.git
   ```

3. **Register it.**
   Pick a local id — `shared-specs` is the natural choice, but if that id is already taken on your machine by an unrelated store, pick something disambiguated (e.g. `brightid-shared-specs`); the id is purely local, it doesn't need to match anyone else's:

   ```bash
   openspec store register /path/to/shared-specs --id shared-specs
   openspec store doctor shared-specs   # verify
   ```

## Wiring up a consuming repo

One-time, per-repo, checked in — independent of the per-machine registration above.
Add to that repo's own `openspec/config.yaml`:

```yaml
schema: spec-driven

references:
  - id: shared-specs
    remote: https://github.com/BrightID/shared-specs.git
```

## Migrating specs (later)

Not done yet.
Per change/spec: the parts describing behavior genuinely shared by 2+ repos move here as the source of truth; each consuming repo keeps only what's specific to its own implementation.
