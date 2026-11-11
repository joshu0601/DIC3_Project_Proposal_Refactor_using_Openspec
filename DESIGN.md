# DESIGN.md — AI Plant Doctor

> Living design document. Started at the Proposal stage (2026-11-03) and kept up to date through the Final Handover.
> Reading order: Problem → Requirements → Architecture → Components → Data → AI/Analytics → Domain Knowledge → Algorithm → Implementation → How to Run → Results → Limitations.
> The full rationale and references are in [`docs/Proposal.md`](docs/Proposal.md). Reference numbers `[n]` below follow that document.

| Status | Stage |
|---|---|
| ✅ | Proposal: architecture, algorithms, core risk models and fusion prototype (with tests) |
| ⏳ | Implementation: IoT node, image model, RAG, LLM planner, LINE/Web UI |
| ⏳ | Experiments and Final Handover |

---

## 1. Problem

People without plant-pathology training (home gardeners, small greenhouse tomato growers in central Taiwan) cannot reliably diagnose leaf problems. Existing AI falls short in three ways:

- **Image-only models fail in the field.** Accuracy goes from 99% on lab images to 31–68% on field images [10][11][12].
- **They ignore the environment.** That contradicts the disease triangle [20], so the models cannot tell look-alike causes apart, such as biotic disease vs. water stress.
- **LLM chatbots give unsupported advice.** They hallucinate region-specific advice [17].

**Goal:** a Domain AI Agent that diagnoses like a plant doctor:
- gathers image, sensor and history evidence;
- reasons over plant-pathology knowledge;
- knows when it is unsure, and then asks follow-up questions or refers the case to a human expert;
- gives IPM-tiered recommendations with citations.

## 2. Requirements

### 2.1 Functional

| ID | Requirement |
|---|---|
| F1 | Accept a text question, leaf photo(s), and plant ID via LINE Bot or Web |
| F2 | Ingest sensor data (T, RH, soil moisture, light, leaf wetness) every 5 min via MQTT |
| F3 | Reject low-quality photos and guide the user to retake them |
| F4 | Produce a ranked differential diagnosis with calibrated confidence, including abiotic causes |
| F5 | Ask up to 2 rounds of follow-up questions when confidence < τ or evidence conflicts; then refer to an expert |
| F6 | Give IPM-tiered recommendations (cultural → physical → biological → chemical); every claim cites a KB source |
| F7 | Proactively alert when accumulated TOMCAST DSV ≥ threshold |
| F8 | Store every interaction and user feedback for evaluation |

### 2.2 Non-functional

| ID | Requirement |
|---|---|
| N1 | Response latency < 10 s (p90) for diagnosis |
| N2 | Explainability: show evidence cards (Grad-CAM, sensor trend, citations) |
| N3 | Safety: no uncited pesticide names or doses (guardrail) |
| N4 | Hardware ≤ NT$1,500 per sensor node |
| N5 | Every component replaceable behind an interface (LLM, image model, vector DB) |
| N6 | Reproducible: pinned dependencies, scripts, and seeds |

## 3. Architecture

![architecture](docs/figures/architecture.png)

```
① User/Scenario ─► ② Query/Input ─► ③ Planner & Reasoning Agent (LLM, ReAct)
                                         │ tool calls          ▲ observations / re-plan
                                         ▼                     │
                     ④ Data Acquisition / Retrieval ─► ⑤ Analytics Engine ─► ⑦ Decision
                        (InfluxDB, CWA API, images,      (5a image, 5b env,      (fusion,
                         history, vector search)          5c risk models)         IPM, τ)
                                         ▲                     ▲                     │
                     IoT node ─MQTT──────┘     ⑥ Domain Knowledge (KG, RAG, rules)  ▼
                                                                         ⑧ Output & Interaction ─► ①
```

The processing flowchart (decisions, loops, exceptions, human-in-the-loop) is in [`docs/figures/flowchart.png`](docs/figures/flowchart.png).

## 4. Components

| # | Component | Responsibility | Interface | Tech (planned) |
|---|---|---|---|---|
| ① | Client | LINE Bot / Web dashboard | HTTPS webhook | LINE Messaging API, FastAPI + HTMX |
| ② | Input gateway | Normalise request, attach plant ID | `Request` dataclass | FastAPI |
| ③ | Planner agent | Intent, decomposition, ReAct tool loop, hypothesis set, follow-ups, guardrails | `PlantDoctorAgent.run(Request) -> Report` | Function-calling LLM (cloud or local Qwen) |
| ④ | Data tools | Sensor window, weather, images, history, KB search | `Tools` protocol (exposable as MCP server) | InfluxDB, CWA Open Data API, S3/MinIO, SQLite |
| ⑤a | Image model | Calibrated top-k disease probabilities + OOD score | `classify_image(bytes) -> (probs, ood)` | timm EfficientNet-B3 / ViT-S, temperature scaling, energy OOD |
| ⑤b/c | Env analytics & risk models | VPD, leaf wetness, TOMCAST DSV, gray-mold index, environmental prior | `environmental_prior(T, RH)` | Python (implemented) |
| ⑥ | Domain knowledge | Disease triangle rules, symptom→cause KG, RAG corpus, IPM rules, thresholds | `search_kb`, `kb_prior` | Chroma + multilingual embeddings, YAML KG |
| ⑦ | Decision | Log-linear evidence fusion, τ-gate, conflict detection, IPM recommendation | `fuse(...) -> Diagnosis` | Python (implemented) |
| ⑧ | Output | Report, evidence cards, alerts, feedback capture | `Report` dataclass | LINE Flex Message, Plotly |

**IoT node:** ESP32 + SHT31 (temperature/humidity) + capacitive soil-moisture sensor + BH1750 (light) + resistive leaf-wetness sensor. MQTT topic `plantdoc/<site>/<node>/telemetry`, JSON payload, 5-min period, deep sleep between readings. Mount the wetness sensor **inside the canopy**: sensor placement changes the risk verdict [24].

## 5. Data

| Data | Source | Use | Split / note |
|---|---|---|---|
| PlantVillage tomato (10 classes) | Hughes & Salathé [29] | Pre-train image model | 80/10/10, lab images |
| PlantDoc tomato subset | Singh et al. [30] (CC BY 4.0) | Fine-tune + **cross-domain test** | Official split |
| TOM2024 tomato subset | Appiah et al. [31] | Cross-domain test (field) | Held out entirely |
| Self-collected field photos | Our greenhouse, target n≈300 | Real-world test, with synced sensor window | Labels: 2 annotators + extension reference |
| Sensor time series | Our IoT node | Environmental prior, alerts, case study | 4–6 weeks |
| Greenhouse climate (public) | WUR Autonomous Greenhouse Challenge [32] | Test the pipeline before our node is deployed | 5-min series |
| Weather | CWA Open Data API | Outdoor context, gap-filling | Cached daily |
| Knowledge base | 植物保護手冊, district research-station fact sheets, papers in Proposal §6 | RAG corpus | Chunk 500 tokens, metadata: crop, disease, source, date |

Raw data is not committed; see [`data/README.md`](data/README.md).

## 6. AI / Analytics and Domain Knowledge

### 6.1 AI / Analytics

- **Image model (5a).**
  - Training: transfer learning (ImageNet → PlantVillage → PlantDoc + field images).
  - Augmentation: background replacement, colour jitter, blur.
  - Calibration: temperature scaling on the validation set.
  - Out-of-distribution: an energy score flags images the model should not trust. Above `ood_limit`, the image weight is set to 0.
  - Explainability: Grad-CAM heatmap for the evidence card.
- **Environmental analytics (5b).**
  - VPD (FAO-56 Tetens [37]).
  - Leaf-wetness hours, from the sensor or the RH ≥ 90% proxy.
  - Soil-moisture trend.
  - Anomaly detection with a rolling z-score.
- **Risk models (5c).**
  - TOMCAST DSV table [22][36], accumulated; alert at 15.
  - Gray-mold heuristic: share of hours at 15–25 °C with RH ≥ 90%, to be calibrated locally.
- **Evidence fusion (⑦).** Log-linear pooling:
  `score(h) ∝ P_img^w1 · P_env^w2 · P_kb^w3` with w = (1.0, 0.6, 0.4) as initial values, tuned on validation.
  - Initial threshold τ = 0.6, later chosen from the coverage–risk curve.
- **LLM planner (③).** Function calling, ReAct loop, max 6 tool calls per turn. Recommendations are generated only from retrieved chunks; a post-check removes uncited claims.

### 6.2 Domain Knowledge → Design Mapping

| Mindset (Proposal §4.1) | Where it lives in the system |
|---|---|
| M1 Disease triangle | `environmental_prior` + fusion: the image alone never decides |
| M2 Symptom ≠ cause / differential diagnosis | Hypothesis set H, follow-up questions, `needs_follow_up` |
| M3 Biotic vs abiotic first | Abiotic hypotheses (`abiotic_water_stress`, …) always in H; VPD and soil-moisture checks |
| M4 IPM | Recommendation prompt enforces tiers; guardrail on chemicals |
| M5 Weather × time epidemiology | DSV accumulation (`TomcastState`) and proactive alerts |

## 7. Algorithm

Algorithm 1 (Agent differential-diagnosis loop) and Algorithm 2 (environmental prior) are given with line numbers in [`docs/Proposal.md` §5.3](docs/Proposal.md#53-pseudocode). Line → code mapping:

| Algorithm lines | Code |
|---|---|
| A1 01–02 | `PlantDoctorAgent.run`: intent via `llm` |
| A1 03–08 | `tools.image_quality_ok`, `tools.classify_image` |
| A1 09–11 | `tools.get_sensor_window` (pre-processing: TODO) |
| A1 12 / A2 01–10 | `domain/risk_models.py::environmental_prior`, `TomcastState` |
| A1 13 | `tools.search_kb`, `tools.kb_prior` |
| A1 14–20 | `decision/fusion.py::fuse` + follow-up / refer branch |
| A1 21–24 | recommendation generation, citation check (TODO), history (TODO) |

## 8. Implementation

```
AIOT_project/
├── README.md, DESIGN.md, requirements.txt, pytest.ini
├── docs/
│   ├── Proposal.md / Proposal.docx / Proposal.pptx
│   ├── figures/ (architecture[_tall].png, flowchart[_wide].png, make_figures.py)
│   └── build/   (build_docx.js: Proposal.md → .docx, build_pptx.js → .pptx)
├── src/plant_doctor/
│   ├── agent/plant_doctor_agent.py   ③ planner skeleton (Algorithm 1)
│   ├── domain/risk_models.py         ⑤b/c VPD, leaf wetness, TOMCAST, gray mold, env prior
│   └── decision/fusion.py            ⑦ evidence fusion, τ-gate, conflict detection
├── tests/test_core.py                unit + end-to-end (stub tools) tests
└── data/README.md                    dataset download & layout
```

Planned additions:
- `iot/firmware/` (ESP32, Arduino)
- `iot/ingest.py` (MQTT → InfluxDB)
- `vision/train.py`
- `kb/build_index.py`
- `app/` (FastAPI + LINE webhook)

## 9. How to Run (Proposal stage)

```bash
python -m pip install -r requirements.txt
python -m pytest            # 7 tests: risk models, fusion, agent end-to-end with stubs
python docs/figures/make_figures.py   # regenerate architecture & flowchart
npm install && npm run build:docx && npm run build:pptx   # regenerate Proposal.docx / .pptx
```

## 10. Results

There are no experimental results yet (Proposal stage). The evaluation protocol is in Proposal §5.4:
- baselines B1–B4 and ablations;
- metrics: Macro-F1, ECE, AURC, citation-support rate, expert score, alert precision/recall, latency, SUS.

## 11. Limitations and Open Questions

- The TOMCAST thresholds and the gray-mold heuristic come from temperate regions [22][36]. They must be recalibrated for Taiwan's hot, humid climate.
- The leaf-wetness RH proxy may over- or under-estimate wetness. Compare it against the physical sensor during the case study.
- The field dataset is small (~300 photos), so per-class results will have wide confidence intervals. Report them with bootstrap CIs.
- The LLM can still mis-state retrieved content. The citation check reduces this risk but does not remove it, so expert review is still needed.
- The system does not prescribe registered pesticide doses. It points to official plant-protection information.

## 12. Change Log

| Date | Change |
|---|---|
| 2026-11-03 | Initial version for Proposal: architecture, requirements, algorithms, risk-model and fusion prototype |
