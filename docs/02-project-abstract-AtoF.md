# 2. Project at a Glance: AI Plant Doctor

## Abstract (A–F)

> A–F used here: **A** Background · **B** Problem/Gap · **C** Objective · **D** Method · **E** Evaluation · **F** Contribution. If the course template defines A–F differently, the six sentences can be relabelled one to one.

- **A. Background.** Home gardeners and small greenhouse tomato growers in central Taiwan often cannot tell what is wrong with a leaf, and plant-protection experts are scarce.
- **B. Problem.** Image-only AI drops from ~99% accuracy on lab images to 31–68% in the field [10][11][12], ignores the environment that the disease triangle says is required [20], cannot separate look-alike causes such as fungal disease vs. water stress, and LLM chatbots hallucinate region-specific advice [17].
- **C. Objective.** Build a Domain AI Agent that diagnoses like a plant doctor: it gathers evidence, reasons over plant-pathology knowledge, knows when it is unsure, and gives safe, cited advice.
- **D. Method.** An LLM planner (ReAct) calls tools that collect three kinds of evidence: a calibrated, OOD-aware leaf-image classifier, an environmental prior computed from a low-cost IoT node (VPD, leaf wetness, TOMCAST DSV, gray-mold index), and a knowledge prior from a symptom→cause graph and RAG corpus. Log-linear pooling fuses them into a ranked differential diagnosis that always includes abiotic causes. A confidence gate τ triggers up to two follow-up rounds and then expert referral. Recommendations follow IPM tiers and are generated only from cited knowledge-base chunks.
- **E. Evaluation.** Cross-domain tests (PlantVillage → PlantDoc / TOM2024 / ~300 own field photos with synced sensor windows), baselines B1–B4 and ablations, with Macro-F1, ECE, AURC, citation-support rate, expert score, alert precision/recall, p90 latency and SUS, plus a 4–6 week greenhouse case study.
- **F. Contribution.** A reproducible, interface-based agent that turns the plant doctor's reasoning (disease triangle, differential diagnosis, biotic-vs-abiotic first, IPM, epidemiology) into explicit system components, showing that environment-aware, uncertainty-aware fusion is more reliable in the field than image-only models or free-form LLMs.

## 5 Cores

| # | Core | What it means | Where |
|---|---|---|---|
| C1 | **Disease-triangle evidence fusion** | Image × environment × knowledge; the image alone never decides | `differential-diagnosis`, D1 |
| C2 | **Differential diagnosis incl. abiotic causes** | A ranked hypothesis set; water stress etc. always present | `differential-diagnosis`, D3 |
| C3 | **Knowing when unsure** | Calibrated probabilities + OOD → τ-gate → ≤ 2 follow-ups → expert referral | `differential-diagnosis`, D2, D4 |
| C4 | **Grounded IPM advice** | RAG-only generation, IPM tier order, citation check, pesticide guardrail | `ipm-recommendation`, D6 |
| C5 | **Proactive epidemiology from low-cost IoT** | ≤ NT$1,500 node, TOMCAST DSV accumulation and gray-mold index, alerts before symptoms | `sensor-telemetry`, `disease-risk-alerts`, D3, D7 |

## 3 Trade-offs

| # | Trade-off | Our choice | Cost we accept | How we manage it |
|---|---|---|---|---|
| T1 | **Coverage vs. risk** (answer more vs. be right) | Selective prediction with τ = 0.6, follow-ups, referral | Some users get "please answer a question / see an expert" instead of an answer | Pick τ from the coverage–risk curve; report AURC |
| T2 | **Interpretable vs. learned fusion** | Log-linear pooling with tunable weights + rule-based risk models | Lower accuracy ceiling than an end-to-end multimodal net | Fits ~300 paired samples; each source can be ablated or switched off (OOD); revisit with more data |
| T3 | **Safety/grounding vs. helpfulness & latency** | Retrieval-only advice, uncited sentences removed, no doses, ≤ 6 tool calls | Shorter, sometimes less complete advice; RAG adds latency against p90 < 10 s | Grow the KB; link to official manual; cache; bound tool loop |

## 10 Details

| # | Detail | Value |
|---|---|---|
| 1 | Fusion formula and initial weights | `score(h) ∝ P_img^1.0 · P_env^0.6 · P_kb^0.4`, tuned on validation |
| 2 | Confidence gate and dialogue bound | τ = 0.6 initially; ≤ 2 follow-up rounds; then referral |
| 3 | Image model | EfficientNet-B3 / ViT-S, ImageNet → PlantVillage → PlantDoc + field; background replacement, colour jitter, blur |
| 4 | Calibration and OOD | Temperature scaling; energy OOD score, image weight = 0 above `ood_limit`; Grad-CAM card |
| 5 | TOMCAST | Daily DSV from wetness hours × mean T [22][36]; accumulate; alert at 15; reset on confirmed action |
| 6 | Gray mold and VPD | Share of hours at 15–25 °C with RH ≥ 90%; VPD by FAO-56 Tetens [37] |
| 7 | Leaf wetness | Physical sensor mounted inside the canopy [24]; fallback proxy RH ≥ 90% |
| 8 | IoT node | ESP32 + SHT31 + capacitive soil + BH1750 + leaf wetness; MQTT `plantdoc/<site>/<node>/telemetry`; 5 min; deep sleep; ≤ NT$1,500 |
| 9 | Agent | Function-calling LLM (cloud or local Qwen), ReAct ≤ 6 tool calls/turn, p90 < 10 s; tools MCP-ready; RAG chunks 500 tokens with crop/disease/source/date |
| 10 | Data and metrics | PlantVillage (pre-train), PlantDoc (fine-tune + cross-domain test), TOM2024 (held out), ~300 own photos (2 annotators); Macro-F1, ECE, AURC, citation-support, expert score, alert P/R, SUS, bootstrap CIs |
