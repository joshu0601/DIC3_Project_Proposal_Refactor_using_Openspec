# Spec Delta

## Purpose

Turns a diagnosis into safe, region-appropriate treatment advice that follows Integrated Pest Management tiers and is grounded in cited knowledge-base sources.

## ADDED Requirements

### Requirement: IPM-tiered recommendations
Recommendations SHALL be ordered by IPM tier: cultural, then physical, then biological, then chemical.

#### Scenario: Fungal disease advice
- **WHEN** the diagnosis is early blight with confidence ≥ τ
- **THEN** the advice lists cultural measures (e.g. remove infected leaves, improve airflow) before any chemical option

### Requirement: Every claim is cited
Every recommendation statement SHALL cite at least one knowledge-base source, and statements without a supporting source SHALL be removed before the reply is sent.

#### Scenario: Uncited sentence generated
- **WHEN** the generated advice contains a sentence not supported by any retrieved chunk
- **THEN** that sentence is removed from the reply

### Requirement: Pesticide guardrail
The system SHALL NOT output a pesticide name or dose unless it appears in a cited knowledge-base source, and SHALL NOT prescribe doses; it SHALL point to official plant-protection information instead.

#### Scenario: Chemical tier requested
- **WHEN** the user asks "要噴什麼藥、多少量?"
- **THEN** the reply names only cited products, gives no dose, and links to the official plant-protection manual

### Requirement: No advice on referral
When a case is referred to an expert, the system SHALL NOT give disease-specific chemical advice.

#### Scenario: Referred case
- **WHEN** the diagnosis outcome is expert referral
- **THEN** the reply contains only general cultural precautions and the referral
