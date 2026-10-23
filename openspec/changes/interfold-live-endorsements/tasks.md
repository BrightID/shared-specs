# Tasks

This is a shared implementation plan, not completed work. App placement and signing must be resolved before UI integration. Philip and Adam's policy decision blocks automatic admission, not collection of evidence.

## 1. Shared contract and decisions

- [ ] 1.1 Review and merge this shared proposal and ADR-0003; verify the accepted documents are linked from AUR-338 and the project tracker.
- [ ] 1.2 Decide whether the evaluator/operator experience is standalone vouch-app or integrated in core Aura; record the chosen repo and integration boundary in shared-specs and Linear before app-local implementation planning.
- [ ] 1.3 Philip and Adam discuss admission separately; verify a linked decision issue lists support thresholds, confidence, disagreement, which answers count, and whether all questions must pass. This change can finish with that policy still open and admission disabled.

## 2. Node protection

- [ ] 2.1 Restrict manager energy routing, normalization, and all role scoring to BrightID; verify regression fixtures with mixed domains, all four role categories, and changed Interfold manager edges leave BrightID scores, levels, and impacts unchanged (AUR-339).
- [ ] 2.2 Test the deployed validator for missing and empty domains; change only missing validation and verify invalid writes store nothing while valid BrightID and Interfold writes still work.
- [ ] 2.3 Verify both protections on aura-test with the regression checks and record the deployed revision before enabling app writes.

## 3. Authenticated app integration

- [ ] 3.1 Test whether an existing Aura signer can submit and confirm an Evaluate on aura-test with real evaluator/subject IDs; record the result, integration path, and revision. If it cannot, complete the signing prerequisite and repeat this check before proceeding.
- [ ] 3.2 After app placement, specify the app-local adapter contract referencing this shared contract; verify it includes exact wire mapping, independent read/write switches, pending/application/failure states, retry, and pre-submit undo (AUR-344).
- [ ] 3.3 Implement the writer without a team, level, or allowlist gate; verify all three questions and both answers use the expected domain/category and integer confidence, unsigned identities cannot write, and failures stay retryable (AUR-345).
- [ ] 3.4 Implement independent Interfold write/read switches with safe practice separation; verify Uniqueness stays mocked, reset makes no node change, and no seed record appears as live evidence (AUR-346).
- [ ] 3.5 Read current inbound Interfold evidence and show evaluator, answer, confidence, and modification time; verify replacements do not duplicate answers, other domains and malformed records are excluded, and the view makes no admission, simulated-level, or unsupported signature claim (AUR-347).

## 4. Integration proof

- [ ] 4.1 With two real test identities on aura-test, submit Yes, confirm it from another client, replace it with No, and submit a different question; verify the latest answers coexist by category, BrightID results stay unchanged, and no Interfold weighted verification is required.
- [ ] 4.2 Disable live capabilities and reset practice; verify applied node evaluations persist and practice remains usable. Record the evidence and limitations in the implementation PRs.

## Workflow follow-up

- Keep AUR-340 and AUR-341 deferred unless the app-placement decision selects their interface; do not let them block this milestone by default.
- After Philip and Adam accept a policy, propose Interfold consumer enforcement and matching Aura presentation in a separate change. Do not implement admission in this one.
- Archive only after the implementation tasks are complete and reviewed. This planning PR is not an implementation completion claim.
