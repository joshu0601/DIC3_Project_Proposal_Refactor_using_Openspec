# Tasks

## 1. Foundation (existing prototype)

- [ ] 1.1 Move the Proposal-stage code (`src/plant_doctor/{agent,domain,decision}`, `tests/test_core.py`) into this repo and verify `python -m pytest` passes all 7 tests
- [ ] 1.2 Define `Tools`, `ImageModel`, `LLM`, `VectorStore` protocols with stub implementations and verify the end-to-end stub test still passes
- [ ] 1.3 Pin dependencies in `requirements.txt` and set global seeds; verify a fresh venv install + pytest succeeds

## 2. Sensor telemetry (`sensor-telemetry`)

- [ ] 2.1 Write ESP32 firmware (SHT31, soil moisture, BH1750, leaf wetness; 5-min publish, deep sleep) and verify readings appear on the MQTT topic
- [ ] 2.2 Implement `iot/ingest.py` (MQTT → InfluxDB) with payload validation; verify tests for valid, malformed and out-of-range payloads
- [ ] 2.3 Implement `get_sensor_window` with gap marking and wetness-source labelling; verify on WUR AGC public data [32]
- [ ] 2.4 Price the node BOM and verify total ≤ NT$1,500 in `iot/README.md`

## 3. Disease-risk alerts (`disease-risk-alerts`)

- [ ] 3.1 Schedule daily DSV update with `TomcastState`; verify accumulation test over a synthetic week
- [ ] 3.2 Send alert at DSV ≥ 15 and reset on user confirmation; verify with a test that mocks the LINE push
- [ ] 3.3 Report gray-mold index in alerts and diagnoses; verify the 60%-humid-hours scenario test

## 4. Image model (`differential-diagnosis`, image evidence)

- [ ] 4.1 Write `data/` download scripts for PlantVillage, PlantDoc, TOM2024; verify split counts match `data/README.md`
- [ ] 4.2 Train `vision/train.py` (ImageNet → PlantVillage → PlantDoc, augmentations); verify Macro-F1 on PlantDoc test is logged
- [ ] 4.3 Add temperature scaling and energy OOD; verify ECE drops on validation and OOD test images are flagged
- [ ] 4.4 Add Grad-CAM output; verify a heatmap PNG is produced for a sample image
- [ ] 4.5 Implement photo-quality check (`diagnosis-intake`); verify blurry/dark test photos are rejected with a retake tip

## 5. Knowledge base and recommendations (`ipm-recommendation`)

- [ ] 5.1 Build `kb/build_index.py` (500-token chunks with crop/disease/source/date metadata into Chroma); verify a known query returns the right fact sheet
- [ ] 5.2 Build YAML symptom→cause KG and `kb_prior`; verify abiotic hypotheses are always returned
- [ ] 5.3 Implement IPM-tier prompt and citation post-check; verify uncited sentences and uncited pesticide names are removed in tests
- [ ] 5.4 Block doses and chemical advice on referred cases; verify the "要噴什麼藥" scenario test

## 6. Planner and diagnosis loop (`differential-diagnosis`)

- [ ] 6.1 Connect the function-calling LLM with a 6-tool-call cap; verify the cap test
- [ ] 6.2 Implement follow-up question generation (≤ 2 rounds) and expert referral; verify low-confidence, conflict and referral scenario tests
- [ ] 6.3 Assemble evidence cards (heatmap, sensor trend, citations) into `Report`; verify the full-evidence scenario test

## 7. Client and logging (`diagnosis-intake`, `interaction-logging`)

- [ ] 7.1 Build FastAPI app with LINE webhook and Web form producing the same `Request`; verify both paths in integration tests
- [ ] 7.2 Render `Report` as LINE Flex Message and HTMX page; verify manually on a phone and a browser
- [ ] 7.3 Store cases, turns and feedback in SQLite by case ID; verify retrieval and label-correction tests

## 8. Integration and evaluation

- [ ] 8.1 Replay 100 requests and verify p90 latency < 10 s
- [ ] 8.2 Run baselines B1–B4 and ablations (w2 = 0, w3 = 0, no τ-gate); verify Macro-F1, ECE, AURC and citation-support rate are reported with bootstrap CIs
- [ ] 8.3 Run the 4–6 week greenhouse case study; verify alert precision/recall, expert score and SUS are recorded and TOMCAST thresholds are re-checked

## Workflow follow-up

- Archive the change with `openspec archive build-plant-doctor-mvp` after the Final Handover review.
- Update `DESIGN.md` Change Log.
