# ADR-0004: Interfold endorsements are answered in apps/vouch, signed with the Aura login

- Status: accepted
- Date: 2026-11-16
- Deciders: Ali. He chose where the flows live (Linear AUR-349) and which identity signs (Linear AUR-350).

## Context

Interfold asks evaluators to answer three questions about a node operator: Keyholder, Self-custody and Disclosure. [ADR-0003](0003-domain-isolation.md) records each answer as an Evaluate operation on the Aura node under domain `Interfold`. It left open which app the operators and evaluators use: the standalone [vouch-app](https://github.com/BrightID/vouch-app) repo, or the Aura monorepo (BrightID/aura).

Since then the vouch practice app has been ported into the Aura monorepo as `apps/vouch` (BrightID/aura PRs #15 to #18). It is a Solid app. `apps/landing`, the site that also serves core Aura, mounts it at `/vouch/` on the same origin. As of this ADR, `apps/vouch` runs only on practice data: a mock adapter and shared practice rooms. It does not sign or send anything to the Aura node.

Core Aura (`apps/core`) already signs BrightID evaluations. It builds the Evaluate operation with `buildEvaluateOperation` in the `@aura/domain` package (`packages/domain/src/operations.ts`), and signs it with the key from the core login: a BrightID login through the BrightID mobile app, or a passkey. Today that function always writes domain `BrightID`, and its category type accepts only BrightID's role categories (subject, player, trainer, manager). It cannot yet write an Interfold answer.

## Decision

Operators and evaluators answer the Interfold questions in `apps/vouch` in BrightID/aura, at `/vouch/` on landing's origin. The standalone vouch-app and the subject-facing `apps/interface` do not host these flows.

An evaluator signs with the identity of their core Aura login. The work to do this is not done yet, and has two parts:

- Core's persisted login store gets a fixed localStorage name. `apps/vouch` then reads the logged-in evaluator's BrightID and secret key from it.
- `buildEvaluateOperation` takes the domain and category as arguments, with `BrightID` as the default domain, so core's existing BrightID evaluations do not change. `apps/vouch` signs its Interfold answers with it.

A person who has not logged in to Aura can still use the practice flows, but cannot send a live answer to the node.

No server signs on an evaluator's behalf. A server-side signing route would let one shared key write answers for many people, and ADR-0003 requires each answer to be signed by its own evaluator. Practice personas, practice-room identities and the mock adapter's session are invented, so they must never supply the evaluator or the key. The live adapter (AUR-344) has to enforce this in code.

## Rationale

The node checks an Evaluate's signature against the evaluator's BrightID signing keys. With core's login, the evaluator is the person's own BrightID, so the node checks the answer against the identity the person already uses across Aura. There is one login for all of Aura. The signer and the operation format already exist and are tested in core, so a live Interfold answer needs only a domain and category argument, not a second signing path. `apps/vouch` already holds the practice flows, so the live flows sit beside them.

## Alternatives considered

- **Standalone vouch-app.** Rejected: a second deployment with its own login and signer. Its unbuilt node adapter planned a server signing route, which this decision rules out.
- **`apps/interface`.** Rejected: the Linear issue AUR-349 notes that this subject-facing app is out of date, and it has no operator or evaluator flows.
- **A passkey identity created inside `apps/vouch`.** Rejected: the evaluator would be a new id made from the passkey's public key, not the person's BrightID. Their answers would not be linked to the BrightID they use elsewhere in Aura.
- **A copy of core's BrightID login inside `apps/vouch`.** Rejected: it gives real BrightIDs, but leaves two login flows to keep in step.

## Consequences

- Core's login store must get a fixed name before `apps/vouch` can read it. Today it is saved without a name, so the library picks a generated key whose value depends on the order the app's code runs.
- `apps/vouch` gains a dependency on `@aura/domain`. Two follow-up issues build on this decision: AUR-344 specifies the live adapter for `apps/vouch`, and AUR-346 adds switches to turn live writes and live reads on separately from practice.
- The standalone vouch-app stays only as a reference for the practice flows.
- Revisit this decision if `apps/vouch` must run on a different origin from core, because it could then no longer read core's login store.
