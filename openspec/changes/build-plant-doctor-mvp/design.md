# Design: AI Plant Doctor MVP

## Context

See `proposal.md` (Why) for motivation and `specs/` for the behavior contract. This design is the OpenSpec version of `DESIGN.md` §3–§7.

Current state (Proposal stage, 2026-11-03):
- Implemented with tests: `domain/risk_models.py` (VPD, leaf wetness, TOMCAST, gray mold, environmental prior) and `decision/fusion.py` (log-linear fusion, τ-gate, conflict detection).
- Skeleton: `agent/plant_doctor_agent.py` (Algorithm 1 with stub tools).
- Not started: IoT node and ingest, image model, RAG index, LINE/Web app.

Constraints: small field dataset (~300 photos), Taiwan's hot humid climate (temperate thresholds need recalibration), NT$1,500 per node, p90 < 10 s, every component replaceable (N5).

## Goals / Non-Goals

**Goals:**
- An 8-stage pipeline (① client → ② gateway → ③ planner → ④ data tools → ⑤ analytics → ⑥ knowledge → ⑦ decision → ⑧ output) where each stage sits behind an interface.
- Interpretable evidence fusion whose weights and threshold can be tuned and ablated.
- A runnable end-to-end path with stub tools at every stage before real components land.

**Non-Goals:**
- End-to-end trained multimodal networks.
- Multi-crop support, actuator control, pesticide-dose prescription.
- Production-grade multi-tenant hosting.

## Architecture

```
① User ─► ② Input gateway ─► ③ Planner agent (LLM, ReAct ≤ 6 tool calls)
                                  │ tool calls            ▲ observations / re-plan
                                  ▼                       │
              ④ Data tools ─► ⑤ Analytics ─────────► ⑦ Decision ─► ⑧ Output ─► ①
              (InfluxDB, CWA,   (5a image, 5b env,     (fusion, τ,
               images, history,  5c risk models)        IPM, guardrail)
               vector search)        ▲
   IoT node ─MQTT─► ④          ⑥ Domain knowledge (KG, RAG corpus, rules)
```

| # | Component | Interface | Tech |
|---|---|---|---|
| ① | Client | HTTPS webhook | LINE Messaging API, FastAPI + HTMX |
| ② | Input gateway | `Request` dataclass | FastAPI |
| ③ | Planner agent | `PlantDoctorAgent.run(Request) -> Report` | Function-calling LLM (cloud or local Qwen) |
| ④ | Data tools | `Tools` protocol (exposable as MCP server) | InfluxDB, CWA Open Data API, MinIO, SQLite |
| ⑤a | Image model | `classify_image(bytes) -> (probs, ood)` | timm EfficientNet-B3 / ViT-S |
| ⑤b/c | Env analytics, risk models | `environmental_prior(T, RH)` | Python |
| ⑥ | Domain knowledge | `search_kb`, `kb_prior` | Chroma + multilingual embeddings, YAML KG |
| ⑦ | Decision | `fuse(...) -> Diagnosis` | Python |
| ⑧ | Output | `Report` dataclass | LINE Flex Message, Plotly |

## Decisions

### D1. Log-linear pooling for evidence fusion (M1 disease triangle)
`score(h) ∝ P_img(h)^w1 · P_env(h)^w2 · P_kb(h)^w3`, initial w = (1.0, 0.6, 0.4), tuned on validation.
- Why: interpretable, each source can be dropped (w = 0) for ablation and OOD handling, works with ~300 field samples.
- Alternatives: (a) end-to-end multimodal network: needs paired image+sensor data we do not have; (b) Bayesian network: needs hand-specified CPTs for every node; (c) LLM decides alone: not calibrated, hallucinates.

### D2. Calibrated image model with energy-based OOD
Transfer learning ImageNet → PlantVillage → PlantDoc + field images; augmentation (background replacement, colour jitter, blur); temperature scaling on validation; energy OOD score; above `ood_limit` set w1 = 0; Grad-CAM for the evidence card.
- Why: fusion needs probabilities that mean what they say; OOD keeps a confidently wrong image from dominating.
- Alternatives: MC-dropout / deep ensembles: better uncertainty but 5–10× inference cost against N1.

### D3. Rule-based environmental prior from published risk models (M3, M5)
VPD (FAO-56 Tetens [37]); leaf-wetness hours from sensor or RH ≥ 90% proxy; TOMCAST DSV [22][36] accumulated with alert at 15; gray-mold index = share of hours at 15–25 °C with RH ≥ 90%; rolling z-score anomaly detection; abiotic hypotheses always in H.
- Why: encodes epidemiology without training data; explainable to growers.
- Alternatives: learned environment→disease model: no labelled time series yet. Thresholds are flagged for local recalibration.

### D4. Selective prediction: τ-gate, ≤ 2 follow-up rounds, then referral (M2)
Initial τ = 0.6, later chosen from the coverage–risk curve (AURC). Conflict = image top-1 and environment top-1 disagree beyond a margin.
- Why: a wrong confident answer is worse than "I am not sure" in plant protection.
- Alternatives: always answer top-1 (higher coverage, higher risk); unlimited dialogue (user fatigue).

### D5. ReAct planner with bounded tool loop
Function calling, max 6 tool calls per turn, tools exposed through the `Tools` protocol (MCP-ready).
- Why: the LLM orchestrates and explains; numeric decisions stay in D1/D4 code.
- Alternatives: fixed pipeline without LLM: cannot handle free-text questions or follow-ups; multi-agent: extra latency.

### D6. Retrieval-only recommendations + citation post-check (M4)
RAG over 植物保護手冊, district research-station fact sheets and papers; chunks of 500 tokens with metadata (crop, disease, source, date). The prompt enforces IPM tiers; a post-check removes sentences with no supporting chunk and blocks uncited pesticide names; no doses.
- Alternatives: fine-tuned agricultural LLM: costly, still unverifiable; free generation: violates N3.

### D7. Low-cost IoT node
ESP32 + SHT31 + capacitive soil moisture + BH1750 + resistive leaf wetness; MQTT topic `plantdoc/<site>/<node>/telemetry`, JSON, 5-min period, deep sleep. Wetness sensor mounted inside the canopy [24].
- Alternatives: commercial weather station: > NT$10,000, violates N4; CWA weather only: misses greenhouse microclimate.

### D8. Interfaces everywhere (N5)
LLM, image model, vector DB and data sources each sit behind a Python protocol with a stub implementation used in tests.

## Algorithm mapping

| Algorithm lines (Proposal §5.3) | Code |
|---|---|
| A1 01–02 | `PlantDoctorAgent.run`: intent via `llm` |
| A1 03–08 | `tools.image_quality_ok`, `tools.classify_image` |
| A1 09–11 | `tools.get_sensor_window` |
| A1 12 / A2 01–10 | `domain/risk_models.py::environmental_prior`, `TomcastState` |
| A1 13 | `tools.search_kb`, `tools.kb_prior` |
| A1 14–20 | `decision/fusion.py::fuse` + follow-up / refer branch |
| A1 21–24 | recommendation generation, citation check, history |

## Risks / Trade-offs

- [Coverage vs. risk at τ] → choose τ from the coverage–risk curve; report AURC.
- [Interpretable fusion caps accuracy vs. learned fusion] → keep D1, revisit when paired data > 1,000 cases.
- [Citation guardrail drops useful but uncited advice; RAG adds latency] → expand KB; cache embeddings; budget ≤ 6 tool calls.
- [Temperate TOMCAST / gray-mold thresholds in Taiwan's climate] → recalibrate during the 4–6 week case study.
- [RH proxy mis-estimates leaf wetness] → compare against the physical sensor.
- [Small field set → wide CIs] → report bootstrap CIs.
- [LLM mis-states retrieved content] → citation check + expert review.

## Migration Plan

Greenfield. Bring components up in order, each replacing its stub: risk models/fusion (done) → IoT + ingest (test first on WUR AGC public data [32]) → image model → KB/RAG → LLM planner → LINE/Web. Rollback = switch the component back to its stub through the interface.

## Open Questions

- Cloud LLM vs. local Qwen: decided by measured latency and cost; does not change specs.
- Exact `ood_limit` and conflict margin: tuned on validation.
