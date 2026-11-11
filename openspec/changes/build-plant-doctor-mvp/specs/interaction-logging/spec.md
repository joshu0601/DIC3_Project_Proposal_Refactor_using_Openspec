# Spec Delta

## Purpose

Keeps a complete record of every diagnosis interaction and user feedback so the system can be evaluated and improved.

## ADDED Requirements

### Requirement: Store every interaction
The system SHALL persist every request, the evidence used, the ranked diagnosis, follow-up turns, recommendations and the referral decision, linked by a case ID.

#### Scenario: Completed case
- **WHEN** a case ends with a diagnosis or a referral
- **THEN** all its turns and evidence references can be retrieved by case ID

### Requirement: Capture user feedback
The system SHALL let the user rate a diagnosis and correct its label, and SHALL store the feedback with the case.

#### Scenario: User corrects diagnosis
- **WHEN** the user marks the diagnosis as wrong and selects the true cause
- **THEN** the correction is stored with the case and available to the evaluation scripts
