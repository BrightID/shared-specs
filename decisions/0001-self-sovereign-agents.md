# ADR-0001: Self-sovereign agents over a shared orchestrator

- Status: accepted
- Date: 2026-07-21
- Deciders: Adam (stated in #agent-workroom), Philip. Ali may amend.

## Context

Three humans (Philip, Adam, Ali) and their agents collaborate on Aura from
different stacks — Claude Code, Augment Intent, TBD. A shared orchestration
platform (Augment Cosmos) could give all agents an instant shared brain via
Augment's context engine.

## Decision

We stay self-sovereign contributors: each human runs their own agent stack and
answers for their own agent. Coordination converges on neutral tools — Discord,
GitHub, Linear.

## Rationale

- Accountability stays per-human: side effects gated by your own human, not a shared queue.
- Vendor neutrality: neutral tools survive any of us switching stacks.
- The record ("decisions are real when a human states them") stays auditable regardless of orchestrator.

## Alternatives considered

- **Augment Cosmos shared runtime** — built for one org coordinating a fleet;
  external-agent support undocumented; blurs per-human approval. Deferred, not
  dismissed: worth a bounded experiment if it can register external agents with
  per-human gates.

## Consequences

We give up the instant shared brain. We compensate with a deliberate portable
context layer — this repo. Revisit if cross-stack coordination friction grows
or Cosmos proves out the experiment.
