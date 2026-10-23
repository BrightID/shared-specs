# Design

## Context

The node's Evaluate path stores one current record per directed evaluator/operator connection and domain/category pair. Its connection reads already return those records. The scorer ignores domain, including while routing manager energy. Vouch-app currently uses invented identities and synchronous mock operations; its live adapter methods throw. See the proposal for the first-release scope and unresolved decisions.

## Goals / Non-Goals

Keep the existing wire format, storage, signature verification, and connection endpoints. Make Interfold evidence usable independently of the scorer. Do not introduce domain administration, evaluator registries, a new scoring algorithm, or a general policy configuration system for this release.

## Decisions

### Keep BrightID scoring restricted to BrightID

Filter both manager energy-routing selections and every scoring selection, including their weight lookups and routing normalization. Preserve BrightID behavior for BrightID-only data. This is smaller than parameterizing the scorer for new domains and avoids choosing Interfold's trust hierarchy.

### Translate at the live application boundary

The app's current lowercase domain, practice aliases, subject category, and question index are internal mock values. The live writer maps them to the Interfold answer encoding spec before signing. Read the node's current records through GET /brightid/v6/users/{id}/connections/inbound and map them into the view. Keep the mock representation private to practice; do not change the node protocol to match it.

### Resolve signing and app placement before wiring the UI

The shared wire contract does not depend on whether vouch-app ships separately or inside core Aura. The implementation depends on that choice and a proven authenticated signer. First test an existing signer against aura-test with real test identities; reuse it if possible. If reuse fails, complete the signing prerequisite before live writes. Do not invent a passkey service, shared signing key, or mobile flow in this proposal. The app-local adapter contract and UI integration follow the app-placement decision.

### Display evidence until admission is decided

Anyone may submit; this permission supplies no independent-person or trust guarantee. Keep identity and confidence visible so Philip and Adam can choose a policy later. Do not implement the demo's two-Yes threshold, No veto, or simulated levels. After policy acceptance, add a separate change specifying enforcement in the Interfold consumer and matching presentation in Aura.

## Risks / Trade-offs

- Open participation allows multiple identities controlled by one person. Raw counts do not establish independent support; automatic admission stays off.
- Existing read endpoints return node records, not a signed admission result. The live view identifies the response as evidence.
- A room reset cannot erase signed node evaluations. Keep live evidence outside the practice world and preserve it across resets.
- Node validation may already reject empty strings in the deployed validator. Measure that first; change only what is missing.

## Rollout and rollback

1. Land and verify the BrightID isolation fix and nonempty-domain behavior on aura-test before enabling a live writer.
2. Resolve app placement and prove the signer; specify the app-local adapter behavior without copying this shared contract.
3. Enable Interfold writes and evidence reads for testing. Verify a real positive answer, replacement by a negative answer, independent questions, and unchanged BrightID results.
4. Release evidence collection with admission explicitly undecided. Philip and Adam's policy work can proceed independently.

To stop a rollout, disable the app's live switches. Keep applied evaluations; do not reset the node or roll back its BrightID isolation protection. Automatic admission has no rollout in this change.
