## Purpose

Let anyone record authenticated answers about Interfold operators and let applications consume current evidence without assuming an admission policy.

## ADDED Requirements

### Requirement: Open endorsement participation

The Aura app SHALL allow anyone with a valid node signing identity to submit Interfold answers without an evaluator allowlist, team membership, or minimum Aura level. Existing protocol signature checks and operation limits SHALL remain in force.

#### Scenario: Evaluator without standing
- **WHEN** an authenticated evaluator with no Aura level or team membership submits a valid Interfold answer within existing operation limits
- **THEN** the app permits submission without an additional participation gate

#### Scenario: Practice persona cannot authorize a live answer
- **WHEN** a selected practice persona has no authenticated signing identity
- **THEN** the app does not submit a live operation as that persona

### Requirement: Interfold answer encoding

The live writer SHALL send Evaluate with domain Interfold, category Keyholder, Self-custody, or Disclosure, evaluation positive for Yes or negative for No, and integer confidence from 1 through 4. The evaluator SHALL sign the exact submitted fields using the existing protocol. The evaluated identifier SHALL identify the intended operator.

#### Scenario: Encode each question
- **WHEN** an evaluator submits any of the three questions with Yes or No and confidence 1, 2, 3, or 4
- **THEN** the signed operation uses the matching exact category and evaluation value, without a question-index field

#### Scenario: Invalid answer input
- **WHEN** the live writer receives an unknown question or a non-integer or out-of-range confidence
- **THEN** it refuses to submit that answer

#### Scenario: Identity binding
- **WHEN** a real evaluator answers about an operator selected in the app
- **THEN** evaluator and evaluated use the real node identifiers rather than practice aliases

### Requirement: Current evidence through existing connection reads

Consumers SHALL obtain current Interfold answers from the operator's inbound connections, select domain Interfold and the three categories in Interfold answer encoding, and retain evaluator identity, answer, confidence, and modification time. They SHALL exclude malformed answers and practice records. A replacement SHALL contribute only the current answer for its evaluator, operator, domain, and category.

#### Scenario: Mixed evidence and replacement
- **WHEN** inbound connections include BrightID records, unknown Interfold categories, and a replaced Interfold answer
- **THEN** the consumer shows only valid current Interfold question answers, without counting the superseded answer

#### Scenario: Invalid stored confidence
- **WHEN** an Interfold record has a confidence outside integer values 1 through 4
- **THEN** the consumer excludes it from the displayed endorsement evidence

### Requirement: No admission decision before policy acceptance

Until Philip and Adam accept an Interfold admission policy, consuming applications SHALL show endorsements without declaring an operator admitted, automatically eligible, or assigned a weighted Aura level. They SHALL NOT describe an unsigned evidence response or a locally calculated summary as a node-signed verification.

#### Scenario: Several Yes answers without an accepted policy
- **WHEN** an operator has multiple Yes endorsements and no admission policy has been accepted
- **THEN** the app displays the evidence and states that admission is not determined

#### Scenario: Conflicting answers
- **WHEN** current endorsements include both Yes and No
- **THEN** both remain visible without applying an invented veto or majority rule

#### Scenario: Unsigned evidence and local summaries
- **WHEN** the view displays an unsigned connection response or a summary calculated by the app
- **THEN** it identifies the data as endorsement evidence and does not claim a node-signed verification

### Requirement: Independent live and practice capabilities

The app SHALL provide separate switches for Interfold writes and endorsement reads, independent of each other and of other capabilities. Live views SHALL contain no seed endorsements. Practice reset SHALL NOT submit changes to node evaluations. Uniqueness practice SHALL remain on the mock.

#### Scenario: Partial live mode
- **WHEN** Interfold evaluation writes and evidence reads are enabled
- **THEN** other capabilities remain in practice unless explicitly enabled, and no practice answer appears as live evidence

#### Scenario: Reset practice after a live answer
- **WHEN** a user resets practice after an Interfold operation has been applied
- **THEN** the node's evaluation remains and Uniqueness practice continues to use mock state

### Requirement: Honest write completion

The app SHALL distinguish pending submission, applied evaluation, and failure. It SHALL honor the existing undo window by submitting only after it expires. Undo after submission SHALL NOT imply that an applied node operation was erased. A failed write SHALL remain retryable without being displayed as applied.

#### Scenario: Undo before submission
- **WHEN** an evaluator undoes an answer within the holding window
- **THEN** no node operation is submitted

#### Scenario: Node rejection
- **WHEN** a submitted operation is rejected or its asynchronous submission fails
- **THEN** the app shows failure and does not mark the answer applied

#### Scenario: Submission is not application
- **WHEN** the node acknowledges submission but has not confirmed application
- **THEN** the answer remains pending until its application status is known

#### Scenario: Retry after a failed write
- **WHEN** an evaluator retries an answer after a confirmed submission failure
- **THEN** the app attempts submission again and marks the answer applied only after application is confirmed

#### Scenario: Undo after submission
- **WHEN** an evaluator tries to undo an answer after it has been submitted
- **THEN** the app does not claim that the submitted or applied evaluation was erased and reports its actual operation status
