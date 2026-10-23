# ADR-0003: An evaluation belongs to its named domain

- Status: proposed
- Date: 2026-10-23
- Deciders: Ali confirmed the first-release scope below on 2026-10-23. Philip and Adam's admission-policy decision remains open. Merging this PR accepts the proposed shared contract.

## Context

The node stores an Evaluate record per domain/category pair on a directed evaluator-to-subject connection. Existing inbound connection reads expose these records. The scorer currently selects the four role categories without filtering domain and publishes results under BrightID. Its manager energy calculation also ignores domain.

Interfold asks three separate questions about an operator. Recording these answers does not require another weighted scoring hierarchy.

## Decision

An evaluation belongs only to the exact domain string on its operation, including case. Missing and empty domains are invalid and cannot fall back to BrightID. The client sends BrightID for Uniqueness and Interfold for Interfold.

Interfold uses the existing Evaluate operation with categories Keyholder, Self-custody, and Disclosure. A Yes maps to positive and a No maps to negative. Confidence is an integer from 1 through 4. No question-index field or second operation is introduced.

The first release records authenticated Interfold endorsements and exposes them through existing connection reads. Anyone with a valid signing identity can endorse, subject to existing protocol validation and operation limits. No evaluator allowlist, team membership, or minimum Aura level is required to submit.

BrightID's entire calculation uses BrightID evidence only. Interfold's weighted scores, energy, and levels are deferred. BrightID evidence cannot be silently reused as Interfold endorsements.

Operator answers happen inside Aura. BrightID mobile needs no answering or domain-signing change (AUR-343, Done). Whether vouch-app is standalone or integrated into core Aura remains open; the older subject-facing interface is not the default target.

## Rationale

The stored domain/category key already separates the three questions. Keeping BrightID scoring restricted to its own domain protects uniqueness while allowing Interfold to collect evidence without choosing a second hierarchy. Open submission preserves participation without confusing an answer with authority to admit an operator.

## Alternatives considered

- **One category plus a question index:** Evaluate has no such field. Confidence represents certainty and cannot encode the question.
- **Missing-domain fallback:** silently converts an incomplete request into BrightID evidence.
- **Weighted Interfold scoring in the first release:** requires decisions about evaluator standing, hierarchy initialization, and levels that are unnecessary for evidence collection.
- **Trusted-evaluator allowlist:** deferred; Ali chose open submission. Any later restriction on which answers count belongs in the admission-policy decision.

## Consequences

The shared behavioral contract, implementation tasks, and open decisions live in [the Interfold OpenSpec change](../openspec/changes/interfold-live-endorsements/proposal.md). This PR changes planning documents only; it does not make the running programs comply.

Philip and Adam must decide the admission policy before automatic admission is built. The app shows current evidence without applying the demo's threshold, No veto, or simulated levels. A locally computed summary is not a node-signed verification.

App placement and a reusable authenticated signer must be resolved before UI integration. The node's existing signature checks remain required. Practice personas cannot authorize live answers, and resetting practice cannot erase applied node evaluations.

The broader design in [How Aura Works](https://github.com/BrightID/foundations/blob/main/how-aura-works.md) remains the direction for future domain scoring. This first release does not claim to implement it.
