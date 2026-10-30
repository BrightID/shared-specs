## Purpose

Keep evaluations in their named question space and protect BrightID results when other Aura applications record answers.

## ADDED Requirements

### Requirement: Explicit domain on evaluation writes

The node SHALL reject an Evaluate operation with a missing or empty domain without storing it or substituting BrightID. Domain names SHALL match exactly, including case.

#### Scenario: Missing or empty domain
- **WHEN** an otherwise valid Evaluate omits domain or supplies an empty string
- **THEN** the operation is rejected and no evaluation is stored

#### Scenario: Exact domain identity
- **WHEN** evaluations name Interfold and interfold
- **THEN** they remain distinct domain values without normalization or fallback

### Requirement: BrightID calculation uses BrightID evidence only

The scorer SHALL use only evaluations whose domain is BrightID throughout energy routing, routing normalization, and all four category calculations. Evaluations in other domains SHALL leave BrightID scores, levels, and impacts unchanged.

#### Scenario: Other-domain manager changes cannot redirect energy
- **WHEN** Interfold manager evaluations are added, replaced, or made negative alongside a fixed BrightID graph
- **THEN** the computed BrightID scores, levels, and impacts equal those computed without the Interfold evaluations

#### Scenario: Matching role categories stay isolated
- **WHEN** a pair has BrightID and Interfold evaluations in each of subject, player, trainer, and manager
- **THEN** only BrightID evaluations contribute to BrightID results

#### Scenario: BrightID compatibility
- **WHEN** the graph contains only valid BrightID evaluations
- **THEN** the scorer retains the existing BrightID calculation and output shape

### Requirement: First release produces no weighted Interfold verification

The scorer SHALL NOT create an Interfold weighted verification for this first release. Interfold answers SHALL remain available as recorded evaluations independently of scoring.

#### Scenario: Interfold answers survive without a scoring hierarchy
- **WHEN** valid Interfold question evaluations are applied and the scorer runs
- **THEN** the evaluations remain readable and no scored Interfold verification is produced by this change
