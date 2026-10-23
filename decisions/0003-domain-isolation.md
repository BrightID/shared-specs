# ADR-0003: An evaluation counts only in the domain on the operation

- Status: proposed
- Date: 2026-10-23
- Deciders: none yet. Merging this pull request accepts the rule. Adam, Philip, or Ali may amend it before then.

## Context

Uniqueness and Interfold share one `Evaluate` operation. On `dev` of [BrightID-Aura-Node](https://github.com/Meta-Node/BrightId-Aura-Node), the node keeps one evaluation per (`domain`, `category`) pair on the directed connection from the evaluator to the evaluated person ([`upsertAuraEval`](https://github.com/Meta-Node/BrightId-Aura-Node/blob/dev/web_services/foxx/v6/db.js#L160)). The schema requires `domain`. When `category` is omitted, the node stores `subject` ([`schemas.js`](https://github.com/Meta-Node/BrightId-Aura-Node/blob/dev/web_services/foxx/v6/schemas.js#L59)). Joi's required string rejects a missing `domain` and accepts `""`.

The scorer selects evaluations whose `category` is `subject`, `player`, `trainer`, or `manager`, and writes that result under the domain name `BrightID`. It does not read `eval.domain` ([`aura.py`](https://github.com/Meta-Node/BrightId-Aura-Node/blob/dev/scorer/verifications/aura.py#L155)). A category outside those four is not added to a BrightID category, and the scorer writes a score under no other domain name.

The Aura player, the client this rule binds, types `domain` as the literal `BrightID` ([`operation_types.d.ts`](https://github.com/BrightID/aura-frontend/blob/main/src/BrightID/api/operation_types.d.ts#L74)).

## Decision

Merging this pull request accepts the rule below. It does not change the scorer or the client. Those programs follow the rule only after the issues under Consequences land.

An evaluation counts only in the `domain` string on that `Evaluate` operation. The string matches exactly, including case. When `domain` is `Interfold`, an `evaluation` of `positive` or `negative` does not change the level or score of any BrightID category. When `domain` is `BrightID`, an `evaluation` of `positive` or `negative` does not change the level or score of any Interfold category. The split holds when the `category` string is the same in both domains.

A missing `domain`, and a `domain` of `""`, are invalid. The node must not store either one, and must not substitute `BrightID`.

For a Uniqueness request, the client sends `domain` `BrightID`. For an Interfold request, the client sends `domain` `Interfold`.

Interfold's three questions are three categories on this same `Evaluate` operation. The category strings are exact:

- `Keyholder`
- `Self-custody` (the hyphen is part of the string)
- `Disclosure`

For an Interfold request, a yes is `evaluation` `positive` and a no is `evaluation` `negative`, and the client sends one of those three strings in `category`. There is no second operation type, and no question-index field.

BrightID categories stay `subject`, `player`, `trainer`, and `manager`. Those four strings are not names of Interfold questions.

## Rationale

The stored key is already (`domain`, `category`). The scorer and the client have to use the domain half of that key. Distinct category names do not replace it. An Interfold answer posted with `domain` `BrightID` and `category` `subject` would change the BrightID subject score, because that is how the scorer selects today.

`Evaluate` stores no question index. `confidence` is the weight of the evaluation. One category per question uses a field the operation already has.

## Alternatives considered

- **One Interfold category plus a question index.** No stored field is an index. Putting the index in `confidence` would replace the weight. Adding a field, or a second operation type, would be a new protocol.
- **Treat a missing or empty domain as BrightID.** A request that omitted `domain` would be stored as Uniqueness. Rejection makes that request fail.

## Consequences

[AUR-339](https://linear.app/aura-brightid/issue/AUR-339/scorer-partitions-evaluations-by-domain) changes the scorer so it counts an evaluation only when `eval.domain` is the domain whose score it is writing, and so `domains[].name` is that same string.

[AUR-340](https://linear.app/aura-brightid/issue/AUR-340/client-sends-the-requested-domain) changes the Aura player so a Uniqueness post sends `domain` `BrightID` and an Interfold post sends `domain` `Interfold`. [AUR-341](https://linear.app/aura-brightid/issue/AUR-341/player-reads-the-active-domain) changes that player so a Uniqueness screen shows only categories stored under `BrightID`, and an Interfold screen shows only categories stored under `Interfold`.

[AUR-343](https://linear.app/aura-brightid/issue/AUR-343/decide-whether-brightid-mobile-signs-the-app-domain) asks whether the BrightID mobile app, a different program from the Aura player, signs an `Evaluate` whose `domain` is the domain a linking app asked for. This rule does not decide that, and it does not put the mobile app under the client rules above.

Today the scorer adds a `subject` evaluation to the BrightID subject score even when `domain` is `Interfold` or `""`. AUR-339 stops that. The node still stores `""`, which this rule forbids, and still stores an omitted `category` as `subject`. After AUR-339, an Interfold post that omits `category` would be stored as Interfold `subject`, which is not one of the three questions. The client has to send the category string. Rejecting `""` is a change in BrightID-Aura-Node. This file does not make that change.
