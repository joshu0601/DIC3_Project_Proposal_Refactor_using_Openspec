# Spec Delta

## Purpose

Lets a grower submit a plant problem (question, leaf photos, plant ID) from LINE or the Web and makes sure only usable photos reach diagnosis.

## ADDED Requirements

### Requirement: Accept multimodal diagnosis requests
The system SHALL accept a diagnosis request containing a text question, zero or more leaf photos, and a plant ID, from either the LINE Bot or the Web dashboard, and normalise both into the same request format.

#### Scenario: LINE request with photo
- **WHEN** a user sends a leaf photo and the text "葉子有黑斑" through LINE with a registered plant ID
- **THEN** the system creates one diagnosis request holding the text, the photo and the plant ID

#### Scenario: Web request without photo
- **WHEN** a user submits only a text question on the Web dashboard
- **THEN** the system accepts the request and marks it as having no image evidence

#### Scenario: Unknown plant ID
- **WHEN** a request references a plant ID that is not registered
- **THEN** the system asks the user to choose or register a plant before diagnosing

### Requirement: Reject low-quality photos
The system SHALL check every submitted photo for blur, exposure and leaf coverage, and SHALL reject photos that fail with specific retake guidance instead of diagnosing them.

#### Scenario: Blurry photo
- **WHEN** a submitted photo fails the blur check
- **THEN** the system replies with a retake tip (e.g. hold steady, move closer) and does not run image diagnosis on it

#### Scenario: Acceptable photo
- **WHEN** a submitted photo passes all quality checks
- **THEN** the photo is forwarded to differential diagnosis
