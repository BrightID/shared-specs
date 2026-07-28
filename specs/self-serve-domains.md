# Self-Serve Domains in Aura
## A Protocol Spec for Creating, Publishing, and Testing New Domains

---

## 1. Overview

Aura today ships one built-in domain of trust — uniqueness — computed by a fixed
evaluation hierarchy (Manager, Trainer, Player/Subject) and operated by teams
whose configuration (seed evaluators, thresholds, evaluation criteria, cadence)
is set centrally. Third-party applications consume the resulting score as a
sybil-resistance guarantee. Everything the protocol currently knows about "what
trust means" is baked into that one domain and into the team settings that run
it.

This document specifies the extension of Aura from a system with one built-in
domain into a self-serve platform on which anyone can **create, publish, and
test their own domain of trust** — food reviews, freelance-marketplace trust,
DAO contributor reputation, local-guide reputation, or anything else — while
continuing to rely on Aura's existing identity layer and graph machinery for
sybil resistance and score propagation.

The central design commitment is that a new domain does not have to be authored
from scratch. **Existing domains and existing team settings are treated as
first-class starting points.** A domain author picks an existing domain (or an
existing team's operational configuration) as a template, copies its role
schema, seed-group shape, scoring parameters, evaluation criteria, and other
defaults into a new domain, and edits from there. Templates are copy-on-create
defaults, not runtime inheritance: once a new domain is published, it evolves
independently of the domain or team it was seeded from, and later changes to
the source do not flow back into the copy.

The authoring lifecycle this doc specifies has three stages:

- **Create.** Draft a domain by forking an existing domain or team settings as
  a template, then edit its role schema, seed group, scoring parameters,
  evaluation criteria, and verification thresholds.
- **Publish.** Register the domain so its attestations begin being counted by
  the scoring engine and its scores become queryable by integrating
  applications.
- **Test.** Exercise the domain against real (or shadow) activity to confirm
  that its configuration behaves as intended before committing to it — either
  in a sandboxed pre-publication mode, or as a published domain whose outputs
  are treated as provisional by consumers.

This produces two layers:

- **Layer 0 — Identity.** The existing uniqueness guarantee, and the existing
  teams that operate it. Unchanged, and continues to underpin sybil resistance
  for everything built on top of it.
- **Layer 1 — Domains.** A new, generalized layer in which any participant can
  register a domain — seeded from an existing domain or team settings when
  desired — publish it to real evaluators, and test its behavior before
  promoting it to general use.

---

## 2. Terminology

| Term | Definition |
|---|---|
| **Identity** | A single verified-unique participant in the Layer 0 network. |
| **Domain** | An application-defined context of trust, with its own role schema, seed group, and scoring configuration. |
| **Seed Group** | The initial set of identities a domain owner designates as trusted at domain creation, from which trust propagates. |
| **Attestation** | A signed record of one identity evaluating another within a specific domain. |
| **Score** | A single continuous trust value, in [0, 1], for a given (identity, domain) pair, derived from the attestation graph. |
| **Tier** | A named label attached to a range of scores within a domain (e.g., Manager, Trainer, Player), if the domain uses named tiers. |
| **Eligibility Threshold** | The minimum score a participant must hold before their attestations are counted by the scoring engine. |
| **Revocation** | Removal of a previously-submitted attestation from score computation. |
| **Dispute** | A flag placed on an attestation pending review, during which it may count at reduced weight. |

---

## 3. Design Principles

### 3.1 Separation of Mechanism and Meaning

| Provided by the protocol (mechanism) | Defined by the domain owner (meaning) |
|---|---|
| One-identity-per-participant guarantee | Role names within the domain |
| Graph structure recording who evaluated whom, when | Number of tiers in the evaluation hierarchy — flat or multi-tier |
| Score propagation from seed identities | The seed group — who is trusted at domain creation |
| Attestation and signature format | Score thresholds for what qualifies as each tier in that domain |
| Sybil resistance (inherited from Layer 0) | Evaluation criteria — what a valid evaluation means |
| Default decay behavior and computational bounds | Domain-specific parameter values within those bounds |

The evaluation hierarchy is not a protocol-level concept — it is fully
configurable per domain. Some domains retain a multi-tier structure resembling
Aura's current Manager/Trainer/Player model; others use a flat, single-tier
structure. The protocol imposes no fixed shape; it only guarantees that whatever
shape is chosen, the resulting scores are computed consistently and resist sybil
attack.

### 3.2 Two Orthogonal Axes: Authorization and Computation

The `role_schema` and the score are two distinct mechanisms that are easy to
conflate, and keeping them separate is what allows a single scoring engine
implementation to serve every domain shape:

- **Authorization** (`role_schema`) answers: *who is permitted to submit an
  attestation that the engine will count?* — e.g., "Trainers may evaluate
  Players," or, in a flat domain, "any participant at or above the eligibility
  threshold may evaluate any other participant."
- **Computation** (score) answers: *given the attestations that were permitted,
  what is this participant's trust value?* — always a single continuous number in
  [0, 1], regardless of how the domain's roles are structured.

Tiers are not a separate scoring mechanism. A tier is a label attached to a band
of the same underlying score — "Trainer" is simply the name given to the range of
scores at or above the Trainer threshold. This is what lets the protocol offer one
scoring engine for every domain, instead of a different computation model per
role shape.

---

## 4. Protocol Architecture

```
Application
    │   registers a domain, embeds SDK, submits/queries attestations
    ▼
Domain Layer
    │   (Domain Registry, Attestation records, per-domain Scoring Engine)
    ▼
Identity Layer   (existing Aura uniqueness network)
```

### 4.1 Identity Layer

Unchanged. Continues to guarantee one identity per real participant. Every
attestation and score computed at Layer 1 is anchored to a Layer 0 identity,
which is what makes domain-level scores sybil-resistant without each domain
needing to solve sybil resistance independently. An identity may link to any
number of domains independently; scores are stored per (identity, domain) pair
with no default interaction between domains (see Section 7).

### 4.2 Domain Registry

```
Domain {
  domain_id:              string, unique
  name:                   string
  description:            string
  role_schema:            RoleSchema | "flat"
  seed_group:             [identity_id]
  scoring_parameters: {
    decay_rate:                     float   // bounded [0, 1], protocol-enforced
    propagation_depth:              int     // bounded [1, 6], protocol-enforced
    min_score_threshold:            float   // bounded [0, 1]
    evaluator_eligibility_threshold: float  // bounded [0, 1] — min score to submit attestations
    dispute_weight:                 float   // bounded [0, 1] — weight of a disputed attestation pending resolution
  }
  verification_thresholds: { tier_name: float }
  evaluation_criteria:    string   // domain-defined description of what a valid evaluation means
  visibility:             "public" | "domain-private" | "participant-controlled"
  created_at, updated_at, config_version
}

RoleSchema {
  tiers: [
    { tier_name: string, evaluates: tier_name | "subjects" }
  ]
}
```

### 4.3 Attestation Layer

```
Attestation {
  attestation_id
  domain_id
  evaluator_id:           identity_id
  subject_id:             identity_id
  score:                  float             // or a structured rating, per domain
  criteria_reference:     optional string   // which evaluation criteria this satisfies
  status:                 "active" | "revoked" | "disputed"
  timestamp
  signature
}
```

### 4.4 Scoring Engine

For each (identity, domain) pair, the engine computes a score through an
iterative propagation process: trust originates at the domain's seed group and
flows outward along active (non-revoked) evaluation edges. Each hop's
contribution is weighted by the evaluator's own current score and attenuated by
the domain's decay rate and propagation depth. Disputed attestations count at
`dispute_weight` until resolved; revoked attestations are excluded entirely.

**Tier assignment is automatic and derived, not manually set.** A participant's
tier is always the highest tier whose `verification_thresholds` value their
current score has crossed. Domain owners do not manually promote participants
between tiers through the normal path — this keeps tier assignment as a pure
function of score rather than a second, parallel source of truth. (A narrow
manual-override path is worth keeping for edge cases — see Section 5.)

**Computational approach.** Recomputing the full propagation graph on every new
attestation does not scale once a domain has more than a small number of
participants. The engine should:
- Recompute incrementally, limiting propagation to the subgraph reachable from
  the changed edge within the domain's `propagation_depth`, rather than the
  entire graph.
- Batch recomputation on a cadence appropriate to the domain's activity level by
  default, with real-time recomputation available as an opt-in for domains
  willing to absorb the added computational cost.

### 4.5 Integration API

```
POST /domains
  { name, role_schema, seed_group, scoring_parameters,
    verification_thresholds, evaluation_criteria, visibility }
  → { domain_id }

GET /domains/{domain_id}
  → current domain configuration

PATCH /domains/{domain_id}
  { scoring_parameters?, verification_thresholds?, evaluation_criteria?, visibility? }
  → updated configuration, config_version incremented
  // Changes to scoring_parameters trigger a full domain recomputation.
  // Changes to verification_thresholds only re-evaluate tier labels, no recomputation needed.

POST /domains/{domain_id}/attestations
  { evaluator_id, subject_id, score, criteria_reference, signature }
  → { attestation_id }

POST /domains/{domain_id}/attestations/{attestation_id}/revoke
  { revoked_by, reason }
  → marks attestation revoked, triggers incremental recomputation of affected scores

POST /domains/{domain_id}/attestations/{attestation_id}/dispute
  { disputed_by, reason }
  → marks attestation disputed; counted at dispute_weight until resolved

GET /domains/{domain_id}/participants/{identity_id}/score
  → { identity_id, domain_id, score, tier, last_updated }
```

---

## 5. Domain Bootstrapping

A domain is empty of meaning at creation — only the seed group holds a nonzero
score. Growth from there follows a fixed sequence:

1. Seed group members evaluate an initial cohort of participants directly.
2. Newly-evaluated participants receive a score computed from their proximity to
   the seed group and the strength of the evaluations they received.
3. A participant becomes eligible to submit attestations that count toward the
   domain's computation once their own score crosses
   `evaluator_eligibility_threshold` (flat domains), or the threshold for a tier
   permitted to evaluate under the domain's `role_schema` (tiered domains).
4. Tier assignment then proceeds automatically as described in Section 4.4.

**Manual override path.** Real domains will occasionally need to inject trust
outside the normal evaluation path — e.g., a domain owner onboarding a
known-credible expert on day one, before that person has accumulated any
attestations. Rather than treating this as a gap, it's worth exposing narrowly:
a domain owner can add an identity directly to the seed group post-launch, but
cannot directly set an arbitrary score for a non-seed participant. This keeps
the one privileged action (expanding the seed group) auditable and distinct
from the normal, attestation-driven scoring path.

---

## 6. Integration Flow

1. **Domain registration.** An application registers a domain: name, role schema
   (or "flat"), seed group, scoring parameters, and verification thresholds.
2. **Identity linking.** A participant links their Aura identity to their
   application account. An identity can link to a given domain at most once.
3. **Evaluation submission.** A trusted evaluator within the domain evaluates
   another participant, producing a signed attestation scoped to that domain.
4. **Score computation.** The scoring engine incrementally recomputes the
   affected participants' domain scores.
5. **Score query.** The application queries a participant's score to determine
   what to unlock.
6. **Continuous update.** Scores are recomputed as further attestations,
   revocations, or dispute resolutions arrive, with updates delivered to the
   application via webhook or polling.

---

## 7. Domain Ownership and Isolation

Each domain is independently owned and fully configured by the application that
registers it. A domain's seed group, scoring parameters, evaluation criteria,
and verification thresholds are set exclusively by that domain's owner, and
domains do not affect one another by default: they are isolated. A single
identity can hold a high score in one domain and no score at all in another,
simultaneously, with no conflict.

This has two direct consequences:

- **Full control.** A domain owner defines exactly what trust means for their
  application without needing to adopt any other domain's standards.
- **Contained failure.** Poor evaluation practices, collusion, or abuse within
  one domain cannot propagate into any other domain's scores, since no domain
  reads from another by default.

What the protocol contributes identically to every domain is the underlying
mechanism: identity, graph structure, attestation format, signature
verification, and score propagation. A new domain owner configures the meaning
of trust for their case on top of infrastructure that already exists, rather
than designing a reputation system from scratch.

---

## 8. Abuse Resistance

Collusion — a cluster of participants evaluating one another favorably to
inflate scores without genuine trust behind them — is the central threat to any
evaluation-based trust system, and the domain-generalized version of the
fake-connection problem Aura already handles at the identity layer.

- **Cluster detection.** Densely mutual-evaluation subgraphs are flagged when
  their internal evaluation density significantly exceeds the domain's
  baseline, and their contribution to propagation is discounted or held for
  review.
- **Evaluator accountability.** An evaluator's own score is affected by the
  downstream outcome of who they vouched for — if a participant they evaluated
  is later found fraudulent or has their attestations revoked, the evaluating
  identity's score takes a corresponding reduction.
- **Revocation.** Any attestation can be revoked, removing it from computation
  and triggering recomputation of everything downstream of it. This is what
  makes evaluator accountability enforceable rather than aspirational — without
  it, there is no mechanism to actually act on a bad evaluation once discovered.
- **Disputes.** A contested attestation is flagged and counted at a reduced,
  domain-configured weight until resolved, rather than either being trusted at
  full weight or removed outright before review.
- **Rate limiting.** A cap, configurable per domain, on the number of
  attestations a single identity can submit within a given time window.
- **Path diversity requirement.** A domain may require that a participant's
  attestations originate from more than one distinct connected component of
  evaluators before reaching full verification, reducing the effectiveness of a
  single colluding cluster acting alone.

---

## 9. Score Decay and Freshness

Trust earned in the past should not necessarily carry the same weight
indefinitely. Decay rate is a domain-level configuration rather than a
protocol-wide constant: a domain built around a stable, slowly-changing property
may set decay close to zero, while a domain built around an actively-maintained
reputation should decay faster, so that inactive participants naturally lose
standing over time rather than retaining a score indefinitely from past
activity.

---

## 10. Architectural Decisions

- **Attestation storage: off-chain by default.** Cost and latency favor
  off-chain storage for most domains. The attestation format is independently
  verifiable (signed, hashable) regardless of storage location, so it can be
  anchored on-chain later — e.g., via periodic Merkle-root commitments — without
  redesigning the format itself.
- **Parameter bounds are protocol-governed; values within them are
  domain-governed.** Each tunable in `scoring_parameters` has a hard min/max
  enforced by the protocol (see Section 4.2); domain owners set the actual value
  within those bounds. This prevents a misconfigured domain from producing
  degenerate or trivially-gameable scores while still leaving real
  configuration flexibility to domain owners.
- **Tier assignment is automatic**, derived purely from score against
  `verification_thresholds` (Section 4.4), with a narrow, auditable manual path
  limited to seed-group expansion (Section 5) — not arbitrary score-setting.
- **Score computation is incremental and batched by default**, with real-time
  recomputation as an opt-in, to keep cost bounded as domains scale (Section
  4.4).

---

## 11. Remaining Open Questions

- **Dispute resolution authority.** Who has final say on a disputed
  attestation — the domain owner, or a protocol-level arbitration path? The
  domain owner is the simpler default; a protocol-level appeal mechanism may be
  worth adding once real dispute volume exists to design against.
- **Configuration versioning.** When a domain owner changes `scoring_parameters`
  materially (e.g., decay rate), should historical scores be fully
  recomputed under the new parameters, or should past attestations remain
  governed by the parameters in effect when they were made? `config_version` is
  tracked in the schema (Section 4.2) but the recomputation policy itself is not
  yet decided.
- **Accountability.** Where a domain's score materially influences a real-world
  decision made through an integrating application, accountability for an
  incorrect or manipulated score should be addressed as part of that domain's
  own terms of integration, rather than assumed to be a protocol-wide guarantee.

---

## 12. Implementation Roadmap

- **Phase 1 — Protocol specification.** Finalize the Domain, Attestation, and
  Score schemas; extend the existing identity layer to support domain-tagged
  records without altering its current uniqueness guarantees.
- **Phase 2 — Reference implementation.** Build the Domain Registry service,
  the Attestation API (including revocation and dispute handling), the
  incremental scoring engine, and a minimal integration SDK covering identity
  linking, evaluation submission, and score querying.
- **Phase 3 — Pilot domains.** Onboard a small number of pilot applications
  spanning different domain shapes — at least one flat-hierarchy domain and one
  multi-tier domain — to validate the configuration model, bootstrapping
  behavior, and abuse-resistance mechanisms under real usage before wider
  rollout.
- **Phase 4 — General availability.** Open self-serve domain registration to
  any application, with the mechanisms from Phases 1–3 in place as defaults.

---

## 13. Appendix: Example Domain Configurations

**A — Local Food Reviews (flat hierarchy)**
```
{
  "name": "local-food-reviews",
  "role_schema": "flat",
  "seed_group": ["<initial trusted local reviewers>"],
  "scoring_parameters": {
    "decay_rate": 0.15,
    "propagation_depth": 3,
    "min_score_threshold": 0.2,
    "evaluator_eligibility_threshold": 0.4,
    "dispute_weight": 0.5
  },
  "verification_thresholds": { "trusted_reviewer": 0.6 },
  "evaluation_criteria": "Evaluator has personally interacted with the subject's reviews and can vouch for their accuracy.",
  "visibility": "public"
}
```

**B — Freelance Marketplace Trust (two-tier)**
```
{
  "name": "freelance-marketplace-trust",
  "role_schema": {
    "tiers": [
      { "tier_name": "verified_client", "evaluates": "subjects" },
      { "tier_name": "senior_freelancer", "evaluates": "verified_client" }
    ]
  },
  "seed_group": ["<platform-vetted senior freelancers>"],
  "scoring_parameters": {
    "decay_rate": 0.05,
    "propagation_depth": 2,
    "min_score_threshold": 0.3,
    "evaluator_eligibility_threshold": 0.5,
    "dispute_weight": 0.4
  },
  "verification_thresholds": { "verified_client": 0.5 },
  "evaluation_criteria": "Evaluator has completed at least one contract with the subject.",
  "visibility": "domain-private"
}
```

**C — DAO Contributor Reputation (multi-tier)**
```
{
  "name": "dao-contributor-reputation",
  "role_schema": {
    "tiers": [
      { "tier_name": "core_maintainer", "evaluates": "core_maintainer" },
      { "tier_name": "core_maintainer", "evaluates": "contributor" }
    ]
  },
  "seed_group": ["<founding core maintainers>"],
  "scoring_parameters": {
    "decay_rate": 0.02,
    "propagation_depth": 4,
    "min_score_threshold": 0.25,
    "evaluator_eligibility_threshold": 0.75,
    "dispute_weight": 0.5
  },
  "verification_thresholds": { "contributor": 0.4, "core_maintainer": 0.75 },
  "evaluation_criteria": "Evaluator has reviewed and merged at least one contribution from the subject.",
  "visibility": "public"
}
```