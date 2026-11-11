# Proposal: Build AI Plant Doctor MVP

## Why

Image-only plant-disease models drop from ~99% accuracy on lab images to 31–68% in the field [10][11][12], ignore the environment (contradicting the disease triangle [20]), and LLM chatbots hallucinate region-specific advice [17]. Growers without plant-pathology training need an agent that diagnoses like a plant doctor: it fuses image, sensor and knowledge evidence, knows when it is unsure, and gives cited IPM advice. The Proposal stage (2026-11-03) delivered the architecture and the risk-model/fusion prototype; this change turns DESIGN.md into a spec-driven build plan for the full MVP.

## What Changes

- Add a LINE Bot / Web intake that accepts text, leaf photos and plant ID, and rejects low-quality photos with retake guidance.
- Add a low-cost IoT sensor node and ingestion pipeline (MQTT → time-series DB, 5-min period).
- Add a differential-diagnosis engine: calibrated image classifier with OOD detection, environmental prior, knowledge prior, log-linear evidence fusion, confidence gate τ, up to 2 follow-up rounds, and expert referral.
- Add IPM-tiered recommendations generated only from retrieved knowledge-base chunks, with a citation guardrail.
- Add proactive disease-risk alerts based on accumulated TOMCAST DSV and a gray-mold index.
- Add interaction and feedback logging for evaluation.

## Capabilities

### New Capabilities

- `diagnosis-intake`: Accepting a diagnosis request (text, photos, plant ID) from LINE/Web and gating photo quality. Covers F1, F3.
- `sensor-telemetry`: Collecting, validating and storing environmental readings from IoT nodes. Covers F2, N4.
- `differential-diagnosis`: Producing a ranked, calibrated differential diagnosis (biotic + abiotic) with evidence cards, follow-up questions and expert referral. Covers F4, F5, N1, N2.
- `ipm-recommendation`: Producing IPM-tiered, citation-backed treatment advice with a pesticide guardrail. Covers F6, N3.
- `disease-risk-alerts`: Proactively warning users when weather-driven disease risk crosses a threshold. Covers F7.
- `interaction-logging`: Persisting every interaction and user feedback for evaluation. Covers F8.

### Modified Capabilities

- None (first change; no existing specs).

## Out of Scope

- Prescribing registered pesticide doses (the system points to official plant-protection information).
- Crops other than tomato; diseases outside the PlantVillage/PlantDoc tomato classes plus listed abiotic causes.
- Automatic actuation (irrigation, ventilation control).

## Impact

- New code: `iot/` (ESP32 firmware, MQTT ingest), `vision/` (training, calibration, OOD), `kb/` (index build), `app/` (FastAPI + LINE webhook), plus existing `src/plant_doctor/{agent,domain,decision}`.
- New dependencies: LINE Messaging API, MQTT broker, InfluxDB, Chroma, timm/PyTorch, a function-calling LLM (cloud or local Qwen).
- External services: CWA Open Data API.
- N5/N6 apply to all new components: interface-based, pinned and seeded.
