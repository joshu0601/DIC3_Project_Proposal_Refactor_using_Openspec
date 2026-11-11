# 1 Introduction (refactored)

> Structure: context → why current AI fails (3 gaps) → key insight → our approach → contributions → paper outline.
> `[n]` = reference numbers in `docs/Proposal.md`. `[new: …]` = reference to add to the list.

## 1.1 Context

Tomato is one of the most widely grown vegetables in Taiwan, both in small greenhouses and in home gardens. When a leaf turns yellow, spots or wilts, growers must decide quickly whether it is a disease, a pest, or simply a watering problem. A wrong call costs a crop or leads to unnecessary pesticide use. Plant-protection experts and district research stations can answer such questions, but they are few, and most home gardeners and small growers have no direct access to them.

## 1.2 Why current AI is not enough

Deep-learning image classifiers reach near-perfect accuracy on curated datasets such as PlantVillage [29][new: Mohanty et al. 2016]. Three gaps prevent them from serving as a plant doctor in practice:

1. **Field robustness.** Accuracy falls from about 99% on lab images to 31–68% on field images with real backgrounds, lighting and multiple leaves [10][11][12]. The models are also over-confident on images they have never seen.
2. **Missing environment.** Plant pathology explains disease through the *disease triangle*: a susceptible host, a pathogen and a favourable environment [20]. Image-only models ignore the third side. As a result they cannot tell apart causes that look alike, for example early blight lesions vs. leaf scorch from water stress, and they cannot warn before symptoms appear.
3. **Unsupported advice.** Large language models (LLMs) can explain and converse, but they produce plausible yet unsupported, region-inappropriate recommendations, including pesticide names and doses [17]. They also do not say when they are unsure.

## 1.3 Key insight

A human plant doctor does not classify a photo. They (i) collect evidence about the plant, its environment and its history, (ii) build a *differential diagnosis* that keeps abiotic causes in view, (iii) ask further questions or send a sample to a lab when the evidence is not decisive, and (iv) recommend Integrated Pest Management (IPM) measures, starting with the least harmful. We argue that an AI assistant should follow the same process, and that each step can be turned into an explicit, testable system component.

## 1.4 Our approach

We present **AI Plant Doctor**, a Domain AI Agent for tomato leaf problems. An LLM planner runs a bounded ReAct tool loop that gathers three kinds of evidence:

- **Image evidence** from a calibrated classifier with an energy-based out-of-distribution (OOD) score;
- **Environmental evidence** from a low-cost IoT node (≤ NT$1,500) that measures temperature, humidity, soil moisture, light and leaf wetness, turned into VPD, leaf-wetness hours, TOMCAST disease-severity values [22][36] and a gray-mold index;
- **Knowledge evidence** from a symptom→cause graph and a retrieval corpus built from the Taiwanese plant-protection manual and research-station fact sheets.

Log-linear pooling fuses the three sources into a ranked differential diagnosis that always contains abiotic hypotheses. A confidence gate τ decides whether to answer, ask up to two rounds of follow-up questions, or refer the case to a human expert. Recommendations are generated only from retrieved sources, ordered by IPM tier, and checked so that every claim is cited and no pesticide dose is prescribed. The same sensor stream also drives proactive alerts when accumulated disease risk crosses a threshold.

## 1.5 Contributions

1. **A domain-grounded agent design** that maps five plant-pathology mindsets (disease triangle, differential diagnosis, biotic-vs-abiotic first, IPM, weather × time epidemiology) onto concrete components and interfaces.
2. **An interpretable, uncertainty-aware fusion method** that combines calibrated image probabilities, an environmental prior and a knowledge prior, with OOD down-weighting and selective prediction (τ-gate, follow-ups, referral).
3. **A low-cost IoT pipeline for proactive alerts** that brings TOMCAST and gray-mold risk models to small greenhouses and home gardens.
4. **A safety layer for LLM advice**: retrieval-only, IPM-tiered generation with a citation check and a pesticide guardrail.
5. **A cross-domain evaluation protocol** (PlantVillage → PlantDoc, TOM2024 and ~300 own field photos with synced sensor windows) measuring accuracy, calibration, coverage–risk, citation support, alert quality, latency and usability, plus an open, reproducible implementation.

## 1.6 Paper outline

Section 2 reviews related work. Section 3 states the requirements and architecture. Section 4 describes the analytics, domain knowledge and fusion algorithm. Section 5 presents the implementation, Section 6 the experiments, and Section 7 discusses limitations and future work.
