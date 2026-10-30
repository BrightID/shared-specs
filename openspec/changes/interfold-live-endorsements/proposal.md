## Decisions

Ali confirmed this first-release scope on 2026-10-23:

1. Record live Interfold endorsements through Aura's existing Evaluate operation and connection reads. Defer weighted Interfold scores, energy, and levels.
2. Anyone who can sign an Evaluate operation for their evaluator identifier under the existing node protocol may endorse. No new participation gate is added; see Open endorsement participation. Recording an answer does not grant operator admission.
3. The endorsement app may be standalone vouch-app or integrated into core Aura; app placement remains open. BrightID mobile needs no answering or domain-signing change.
4. Protect BrightID's complete scoring calculation from evaluations in other domains before enabling live Interfold writes.

**Still open:**

- Philip and Adam decide the acceptance policy: which answers count toward admission, the number of supporters, confidence requirements, treatment of No answers, and whether all three questions must pass. Open participation does not settle these questions. Until a policy is accepted, show evidence without an automatic admission result.
- Decide whether the operator/evaluator experience is a standalone vouch-app or part of the core Aura app. The existing subject-facing interface is not the assumed integration target. App placement blocks UI integration, not the shared node contract.
- Test whether the chosen app can sign Evaluate operations for evaluator identifiers under the existing node protocol and address the intended operator. Reuse an existing signing implementation if possible; otherwise implement that integration before enabling writes. No new authentication mechanism is proposed, and a practice alias alone cannot sign an operation.

## Why

Interfold needs real answers about node operators. The node already stores evaluations by domain and category, but its scorer pools the existing role categories into BrightID. A first release can record and expose Interfold evidence without building another scoring hierarchy.

## What Changes

- Specify the shared Interfold endorsement contract and full protection of BrightID scoring.
- Use domain `Interfold` and categories `Keyholder`, `Self-custody`, and `Disclosure`, as proposed in ADR-0003.
- Read current endorsements through the existing inbound-connections API. Keep practice data separate from live evidence.
- Specify how the app reports pending writes, failures, and the absence of an accepted admission policy.

## Capabilities

### New Capabilities

- `domain-isolation`: domain validation and isolation of the complete BrightID calculation.
- `interfold-endorsements`: open submission of signed Interfold evaluations and consumption of their current stored records.

### Modified Capabilities

None. This change does not modify an existing published OpenSpec capability.

## Impact

BrightID-Aura-Node needs scoring filters and a verified nonempty-domain validation rule. The chosen endorsement app needs operation signing, live evaluation writes and connection reads. Interfold consumes those records; admission enforcement awaits Philip and Adam's policy. No new operation, endpoint, collection, package dependency, or BrightID mobile work is planned. The older subject-facing Aura player's write/read tasks, AUR-340 and AUR-341, are deferred unless the app-placement decision selects that interface.
