# Self-Serve Domains in Aura
## A Protocol Spec for Creating, Publishing, and Testing New Domains

---

## 1. Overview

Aura today ships one built-in domain of trust — uniqueness — computed by a fixed
evaluation hierarchy (Manager, Trainer, Player/Subject) and operated by one or
more teams whose configuration (team owners, thresholds, evaluation
criteria, cadence) is set centrally. Each team computes scores for its
participants independently; third-party applications typically consume an
aggregate across teams as a sybil-resistance guarantee. Everything the protocol
currently knows about "what trust means" is baked into that one domain and into
the team settings that run it.

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
schema, scoring parameters, verification thresholds, evaluation criteria, and
other defaults into a new domain, and edits from there. Templates are copy-on-create
defaults, not runtime inheritance: once a new domain is published, it evolves
independently of the domain or team it was seeded from, and later changes to
the source do not flow back into the copy.

The authoring lifecycle this doc specifies has three stages:

- **Create.** Draft a domain by forking an existing domain or team settings as
  a template, then edit its role schema, scoring parameters, evaluation
  criteria, and verification thresholds; when creating the domain, also
  designate the initial team's owners, who become that team's first managers
  and the starting point of its manager-energy calculation (Section 4.4).
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
| **Domain** | An application-defined context of trust, with its own role schema and scoring configuration, operated by one or more teams (see Section 4.6). |
| **Attestation** | A signed record of one identity evaluating another within a specific domain. |
| **Score** | A single continuous trust value, in [0, 1], for a given (identity, domain) pair, derived from the attestation graph. |
| **Tier** | A named label attached to a range of scores within a domain (e.g., Manager, Trainer, Player), if the domain uses named tiers. |
| **Eligibility Threshold** | The minimum score a participant must hold before their attestations are counted by the scoring engine. |
| **Revocation** | Removal of a previously-submitted attestation from score computation. |
| **Dispute** | A flag placed on an attestation pending review, during which it may count at reduced weight. |
| **Team** | An operational configuration of a domain: a set of team owners, scoring parameters, evaluation-criteria interpretation, and verification thresholds. A domain may be operated by one or more teams; each team computes scores independently (see Section 4.6). |
| **Team Settings** | The team-scoped configuration — team owners, scoring parameters, evaluation criteria, verification thresholds — that a new team or new domain may copy as a starting point, distinct from the domain-level role schema shared across all teams operating that domain. |
| **Team Owner** | An identity that bootstraps and administers a team. Team owners become the first managers of their team (the initial members of its top evaluation tier), can add or remove other owners by a 2/3 majority vote among current owners, and expand the team by evaluating other members. |
| **Manager Energy** | A distinct calculation used for the top evaluation tier when a domain uses a multi-tier role schema. Energy originates at team owners, iterates a small bounded number of times, and is redistributed — not added — by positive top-tier evaluations (see Section 4.4). |
| **Aggregate Score** | An app-facing score for an identity in a domain, derived by combining that identity's team-local scores across the teams operating the domain. Aggregation is a consumer-side or league-side concern; the protocol computes per-team scores that aggregation builds on. |
| **Provisional Participation** | A pre-eligibility mode in which a participant may submit attestations that are recorded but do not yet contribute to any team's scoring, used to bootstrap involvement before a team's owners have evaluated them. |

---

## 3. Design Principles

### 3.1 Separation of Mechanism and Meaning

| Provided by the protocol (mechanism) | Defined by the domain owner (meaning) |
|---|---|
| One-identity-per-participant guarantee | Role names within the domain |
| Graph structure recording who evaluated whom, when | Number of tiers in the evaluation hierarchy — flat or multi-tier |
| Score propagation from designated origin identities | The initial team's owners — who are trusted at team creation and serve as the origin for score propagation |
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

A domain has one or more **teams** operating it (see Section 4.6). A team is an
operational configuration of the domain — its team owners, scoring parameters,
and verification thresholds — not a separate domain. The domain-level
`scoring_parameters` and `verification_thresholds` below define the defaults a
team inherits when it is registered on the domain; the initial team's owners
are supplied at domain registration (see Section 4.5) and every team carries
its own team-scoped values thereafter.

```
Domain {
  domain_id:              string, unique
  name:                   string
  description:            string
  role_schema:            RoleSchema | "flat"
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

Team {
  team_id:                    string, unique within a domain
  domain_id:                  string
  team_owners:                [identity_id]         // owners who bootstrap and administer this team
  scoring_parameters:         ScoringParameters     // may override domain defaults within protocol bounds
  verification_thresholds:    { tier_name: float }
  manager_energy_iterations:  int                   // small bounded integer, applies only when role_schema has a top tier
  created_at, updated_at, config_version
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

For each (identity, domain, team) tuple, the engine computes a score through
an iterative propagation process: trust originates at the team's owners and
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

**Multi-team scoring.** In a domain with multiple teams (Section 4.6), the
propagation process described above runs independently for each team, using
that team's own owners and (possibly overridden) scoring parameters, producing
a per-team score for the participant. In single-team domains, this distinction
collapses.

**Manager energy — a distinct calculation for the top tier.** When a domain's
role schema includes a top evaluation tier (historically called "Manager"),
members of that tier are scored not by the normal propagation path above but
by a bounded, redistributive calculation referred to as *manager energy*:

- **Seeded exclusively at team owners.** The team's owners are the sole source
  of manager energy for their team, so a team's top tier is bootstrapped
  without requiring prior attestations to it.
- **Bounded iterations.** A small, team-configured number of weighted-
  SybilRank-style iterations (typically 2–4 depending on team size), so cost
  stays predictable as teams grow.
- **Redistributive rather than additive.** A positive top-tier evaluation
  redirects a share of the evaluator's outbound energy; issuing an additional
  positive evaluation reduces the energy flowing to the evaluator's already-
  vouched-for peers. This is a deliberate departure from Trainer/Player-style
  scoring, where one evaluation does not diminish others.

The remaining tiers (Trainer, Player, etc., where present) continue to score
by the normal propagation path.

### 4.5 Integration API

```
POST /domains
  { name, role_schema, initial_team_owners, scoring_parameters,
    verification_thresholds, evaluation_criteria, visibility }
  → { domain_id, initial_team_id }
  // `initial_team_owners` seeds the domain's initial team; additional teams
  // are registered via POST /domains/{domain_id}/teams below.

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
  → { identity_id, domain_id, score, tier, per_team: [ { team_id, score, tier } ], last_updated }
  // `score` and `tier` are the domain's default aggregation across teams (Section 4.6);
  // consumers can also compute their own aggregate from `per_team`.

POST /domains/{domain_id}/teams
  { team_owners, scoring_parameters?, verification_thresholds?, manager_energy_iterations? }
  → { team_id }
  // Omitted fields fall back to the domain's defaults from Section 4.2.

GET /domains/{domain_id}/teams
  → [ Team, ... ]   // per Team schema in Section 4.2
```

### 4.6 Teams and Aggregate Outputs

A domain may be operated by one or more **teams**. A team is an operational
configuration of a domain (Section 4.2), not a separate domain. Every team
operating a given domain works from the same domain-level role schema and
evaluation-criteria description, but with its own team owners, scoring
parameters, and verification thresholds.

Two consequences follow:

- **Team-local scores and tiers are independent.** A participant may hold
  different scores — and therefore different tier assignments — in different
  teams within the same domain, without conflict. The scoring engine (Section
  4.4) runs once per (identity, domain, team) tuple, not once per (identity,
  domain). Team owners define team-specific thresholds and verification
  requirements, so what qualifies as e.g. a "Trainer" in one team is not
  necessarily the same in another.
- **App-facing outputs are aggregates.** Integrating applications typically
  consume a single aggregate score per (identity, domain), combining the
  participant's team-local scores across the teams operating the domain.
  Aggregation — which teams to include, and how to weight them — is
  deliberately a consumer-side or league-side concern rather than a protocol
  guarantee, so that consumers retain the ability to include, exclude, or
  reweight a specific team without the protocol having to arbitrate that
  decision.

Attestation submission carries team scope: an evaluator may evaluate the same
subject only once per role, but that evaluation can be applied to any of the
teams the evaluator belongs to (extending the base attestation schema in
Section 4.3, whose team-scoping fields are elided for readability).

A single-team domain is the degenerate case: its team-local scores are its
app-facing scores. The multi-team case is what motivates the aggregation layer
and the resilience properties described in Section 7.

---

## 5. Domain Bootstrapping

A domain is empty of meaning at creation — only the initial team's owners hold
a nonzero score. Growth from there follows a fixed sequence:

1. Team owners evaluate an initial cohort of participants directly.
2. Newly-evaluated participants receive a score computed from their proximity to
   the team's owners and the strength of the evaluations they received.
3. A participant becomes eligible to submit attestations that count toward the
   domain's computation once their own score crosses
   `evaluator_eligibility_threshold` (flat domains), or the threshold for a tier
   permitted to evaluate under the domain's `role_schema` (tiered domains).
4. Tier assignment then proceeds automatically as described in Section 4.4.

**Manual override path.** Real domains will occasionally need to inject trust
outside the normal evaluation path — e.g., a team onboarding a known-credible
expert on day one, before that person has accumulated any attestations. Rather
than treating this as a gap, it's worth exposing narrowly: existing team owners
can add another identity as a team owner post-launch (per the 2/3 majority rule
in Section 2), but cannot directly set an arbitrary score for a non-owner
participant. This keeps the one privileged action (expanding the team's owner
set) auditable and distinct from the normal, attestation-driven scoring path.

**Starting from existing settings.** A new domain does not need to be authored
from scratch, and neither does a new team on an existing domain. Two copying
paths are first-class:

- **Copy a domain's role schema and defaults.** When creating a new domain, its
  author may seed the new domain by copying an existing domain's role schema,
  evaluation-criteria description, and scoring-parameter defaults. The copy is
  applied at create time; the new domain evolves independently thereafter.
- **Copy a team's settings.** When creating a domain, or registering an
  additional team on an existing domain, the author may copy an existing team's
  settings — team-owner set, scoring parameters, evaluation criteria,
  verification thresholds — as starting defaults. This gives a new team a
  known-working configuration to iterate from rather than starting from
  protocol defaults.

Both copies are point-in-time snapshots, not live inheritance. Later changes to
the source domain or team do not flow back into a copy.

**Provisional participation.** Anyone may begin submitting attestations in a
domain before any team's owners have evaluated them. Provisional
attestations are recorded but do not contribute to any team's scoring; they
become eligible to count only once the submitter crosses the relevant team's
`evaluator_eligibility_threshold`. This lets participation begin naturally in
advance of formal admission to a team, without inflating scores in the interim.

---

## 6. Integration Flow

1. **Domain registration.** An application registers a domain: name, role schema
   (or "flat"), and the initial team's settings — team owners, scoring parameters,
   verification thresholds. Additional teams may be registered on the domain
   later (Section 4.5).
2. **Identity linking.** A participant links their Aura identity to their
   application account. An identity can link to a given domain at most once.
3. **Evaluation submission.** A trusted evaluator submits a signed attestation
   scoped to the domain; in multi-team domains, the attestation applies to
   whichever teams the evaluator both belongs to and elects to include
   (Section 4.6).
4. **Score computation.** The scoring engine incrementally recomputes the
   affected participants' team-local scores.
5. **Score query.** The application queries a participant's aggregate score for
   the domain (per Section 4.5), optionally inspecting the per-team breakdown
   to apply its own team-selection or weighting policy.
6. **Continuous update.** Scores are recomputed as further attestations,
   revocations, or dispute resolutions arrive, with updates delivered to the
   application via webhook or polling.

---

## 7. Isolation, Resilience, Privacy, and Accountability

### 7.1 Domain Isolation

Each domain is independently owned and fully configured by the application that
registers it. A domain's role schema, scoring parameters, evaluation criteria,
and verification thresholds — along with the initial team's owners — are set
exclusively by that domain's owner, and domains do not affect one another by
default: they are isolated. A
single identity can hold a high score in one domain and no score at all in
another, simultaneously, with no conflict.

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

### 7.2 Multi-Team Resilience

Because a domain may be operated by more than one team (Section 4.6), the
domain does not depend on any single team being healthy for its app-facing
outputs to remain useful:

- **Team-level compromise stays team-level.** If a team's owners are captured,
  or its evaluations become systematically biased or collusive,
  applications and aggregators can exclude that team from their aggregate
  without withdrawing from the domain. The other teams operating the domain
  continue to compute team-local scores unaffected, so the domain as a whole
  keeps functioning.
- **Recovery is additive.** New teams can be registered on the domain and
  bootstrapped by copying a healthy team's settings (Section 5), so the
  response to a compromised team is to add or reweight teams rather than
  requiring a protocol-level takeover of the affected one.
- **Aggregation is the consumer's lever.** Because aggregation across teams is
  a consumer-side or league-side decision (Section 4.6), the mechanism for
  excluding a team already exists where the app-facing score is computed —
  without the protocol having to formalize a removal path.

### 7.3 Privacy

The protocol inherits Layer 0's privacy posture: no information about a
participant should be disclosed to a party that does not already know it.
Because evaluations rely on evaluators who already know the participants they
evaluate, the protocol does not create a new class of counterparties that must
be handed personal data in order to verify a participant. Domain owners may
still choose different visibility settings for their domain's attestations and
scores (`public`, `domain-private`, `participant-controlled`; see Section 4.2),
but no visibility setting overrides the Layer 0 rule for the underlying
identity data.

### 7.4 Accountability

Evaluators have skin in the game. An evaluator's own score is affected by
downstream outcomes on who they vouched for (Section 8), and attestations can
be revoked or disputed after the fact. In practice this means that a
consistently poor or captured evaluator sees their own score drop quickly to
a point where their attestations stop influencing the domain's outputs —
without requiring an out-of-band ban list or a protocol-level punishment
mechanism.

Accountability for how an app uses a domain's scores — including the choice of
which teams to include in its aggregate and how to weight them — sits with the
integrating application rather than the protocol; see Section 11.

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
  limited to team-owner expansion (Section 5) — not arbitrary score-setting.
- **Score computation is incremental and batched by default**, with real-time
  recomputation as an opt-in, to keep cost bounded as domains scale (Section
  4.4).
- **Multi-team aggregation is a consumer-side concern.** The protocol computes
  team-local scores; combining them into an app-facing aggregate — and choosing
  which teams to include or exclude — is left to consumers or leagues (Sections
  4.6, 7.2), so applications retain a natural lever for handling compromised
  teams without protocol-level arbitration.
- **Privacy is inherited from Layer 0, not re-invented at the domain layer.**
  Domain-level visibility settings tune how attestations and scores are
  exposed but do not weaken the Layer 0 rule against sharing identity data
  with parties that don't already know the participant (Section 7.3).

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

- **Phase 1 — Protocol specification.** Finalize the Domain, Team, Attestation,
  and Score schemas; extend the existing identity layer to support domain-tagged
  records without altering its current uniqueness guarantees.
- **Phase 2 — Reference implementation.** Build the Domain Registry service
  (including team registration), the Attestation API (including revocation and
  dispute handling), the incremental scoring engine (including the manager-
  energy calculation for multi-tier domains), and a minimal integration SDK
  covering identity linking, evaluation submission, and score querying.
- **Phase 3 — Pilot domains.** Onboard a small number of pilot applications
  spanning different domain shapes — at least one flat-hierarchy domain, one
  multi-tier domain, and one multi-team domain — to validate the configuration
  model, bootstrapping behavior, aggregation behavior, and abuse-resistance
  mechanisms under real usage before wider rollout.
- **Phase 4 — General availability.** Open self-serve domain registration to
  any application, with the mechanisms from Phases 1–3 in place as defaults.

---

## 13. Appendix: Example Domain Configurations

**A — Local Food Reviews (flat hierarchy)**
```
{
  "name": "local-food-reviews",
  "role_schema": "flat",
  "initial_team_owners": ["<initial trusted local reviewers>"],
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
  "initial_team_owners": ["<platform-vetted senior freelancers>"],
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
  "initial_team_owners": ["<founding core maintainers>"],
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