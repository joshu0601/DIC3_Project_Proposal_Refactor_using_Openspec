# Spec Delta

## Purpose

Produces a plant-doctor-style differential diagnosis: a ranked, calibrated list of biotic and abiotic causes built from image, environment and knowledge evidence, and knows when to ask more or refer to a human.

## ADDED Requirements

### Requirement: Ranked differential diagnosis
The system SHALL return a ranked list of candidate causes with calibrated confidence for each request, combining image, environmental and knowledge evidence so that no single source decides alone.

#### Scenario: Image and environment agree
- **WHEN** the image strongly suggests early blight and the sensor window shows long leaf-wetness hours at 20–28 °C
- **THEN** early blight is ranked first with confidence ≥ τ and both evidence sources are shown

#### Scenario: No photo
- **WHEN** the request has no usable photo
- **THEN** the diagnosis is built from environment and knowledge evidence only and its confidence reflects the missing image

### Requirement: Abiotic causes always considered
The differential diagnosis SHALL always include abiotic hypotheses (at least water stress) alongside biotic diseases.

#### Scenario: Dry soil with wilting
- **WHEN** soil moisture has been below the dry threshold for 48 h and VPD is high
- **THEN** abiotic water stress appears in the ranked list even if the image model favours a disease

### Requirement: Untrusted image evidence is ignored
The system SHALL detect out-of-distribution photos and SHALL give such photos zero weight in the diagnosis.

#### Scenario: Non-leaf photo
- **WHEN** the photo's OOD score exceeds the OOD limit
- **THEN** the image contributes nothing to the ranking and the report says the photo could not be trusted

### Requirement: Confidence gate with follow-up questions
When the top confidence is below τ or evidence sources conflict, the system SHALL ask targeted follow-up questions, for at most 2 rounds per case.

#### Scenario: Low confidence
- **WHEN** the top hypothesis has confidence < τ (initially 0.6)
- **THEN** the system asks a follow-up question chosen to separate the top hypotheses

#### Scenario: Conflict between image and environment
- **WHEN** the image favours a fungal disease but the environment strongly disfavours it
- **THEN** the system flags the conflict and asks a follow-up question

### Requirement: Expert referral
After 2 follow-up rounds without reaching τ, the system SHALL stop guessing and refer the case to a human expert with a case summary.

#### Scenario: Still unsure after 2 rounds
- **WHEN** confidence is still < τ after the second follow-up round
- **THEN** the system returns a referral with the evidence summary and does not present a single definitive diagnosis

### Requirement: Explainable evidence cards
Each diagnosis SHALL include evidence cards: an image heatmap when a photo was used, the relevant sensor trend, and the knowledge sources used.

#### Scenario: Full evidence
- **WHEN** a diagnosis uses a photo, sensor data and KB chunks
- **THEN** the report contains a heatmap card, a sensor-trend card and a citation card

### Requirement: Diagnosis latency
The system SHALL return a diagnosis (or a follow-up question) within 10 seconds at the 90th percentile.

#### Scenario: Latency benchmark
- **WHEN** 100 representative requests are replayed
- **THEN** the p90 end-to-end latency is < 10 s
