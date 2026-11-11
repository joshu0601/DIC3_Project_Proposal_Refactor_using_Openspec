# 2 Related Work (refactored)

> Organised by the four building blocks of our system; each subsection ends with **the gap we address**.
> `[n]` = reference numbers in `docs/Proposal.md`. `[new: …]` = well-known paper to add to the reference list. `[REF?]` = claim needs a citation from the Proposal list.

## 2.1 Image-based plant disease recognition

CNNs trained on PlantVillage [29] reported near-perfect accuracy on 38 crop–disease classes [new: Mohanty et al. 2016], and later work compared many architectures on similar lab data [new: Ferentinos 2018]. Because PlantVillage images show single leaves on uniform backgrounds, these results overestimate real performance. PlantDoc [30] and TOM2024 [31] collected field images with cluttered backgrounds, and cross-domain studies show accuracy dropping to 31–68% [10][11][12]. Proposed remedies include transfer learning from field data, background replacement and stronger augmentation, and Vision Transformers [REF?].

**Gap.** These systems still output a label from one image. They are not calibrated, cannot detect images outside their training distribution, and have no access to the environment. We use the image model as *one calibrated, OOD-aware evidence source* rather than the decision maker.

## 2.2 Environment-driven disease forecasting and agricultural IoT

Plant epidemiology links infection to weather through the disease triangle [20]. Forecasting systems such as TOMCAST compute daily disease-severity values from leaf-wetness duration and temperature and recommend spraying when the accumulated value crosses a threshold [22][36]; similar humidity-based rules exist for gray mold (*Botrytis cinerea*) [REF?]. Agricultural IoT studies deploy low-cost sensor nodes for greenhouse monitoring [REF?], and sensor placement inside vs. outside the canopy is known to change the risk estimate [24]. Public greenhouse datasets such as the WUR Autonomous Greenhouse Challenge [32] provide climate series for testing.

**Gap.** Forecasting models give field-level *risk* but not a *diagnosis* of a specific plant, and IoT dashboards show numbers without interpretation. Few systems combine sensor-based risk with leaf images. We turn the risk models into an *environmental prior* inside the diagnosis and also use them for proactive alerts.

## 2.3 LLMs, retrieval and agents for agriculture

LLMs can pass agricultural exams [new: Silva et al. 2023] and are used as farm advisory chatbots, but they hallucinate and give advice that does not fit local regulations [17]. Retrieval-augmented generation (RAG) grounds outputs in documents [new: Lewis et al. 2020], and the ReAct pattern lets an LLM interleave reasoning with tool calls [new: Yao et al. 2023]. Agricultural RAG systems typically answer text questions over manuals [REF?], and multimodal LLMs can describe leaf photos but are not calibrated plant-disease classifiers [REF?].

**Gap.** Existing agricultural LLM systems answer questions; they do not run a structured differential diagnosis, do not use live sensor data, and rarely enforce citation or pesticide-safety rules. Our agent uses the LLM to *plan and explain*, keeps numeric decisions in calibrated code, and adds a citation check and IPM-tier guardrail.

## 2.4 Uncertainty, selective prediction and human-in-the-loop

Modern neural networks are over-confident; temperature scaling is a simple and effective post-hoc fix [new: Guo et al. 2017]. Energy scores detect out-of-distribution inputs better than softmax confidence [new: Liu et al. 2020]. Selective classification lets a model abstain to trade coverage for risk, evaluated with risk–coverage curves [new: Geifman & El-Yaniv 2017], and learning-to-defer methods route hard cases to human experts [new: Mozannar & Sontag 2020]. These ideas are rarely applied in plant-disease tools [REF?].

**Gap.** Prior work studies abstention for a single model. We apply it to a *multi-source* diagnosis and replace plain abstention with a dialogue: targeted follow-up questions first, expert referral second.

## 2.5 Positioning

| Approach | Image | Environment | Knowledge / citations | Uncertainty & referral | Proactive alert |
|---|:-:|:-:|:-:|:-:|:-:|
| Image classifiers (§2.1) | ✓ | – | – | – | – |
| Disease forecasting / IoT (§2.2) | – | ✓ | partial (rules) | – | ✓ |
| Agricultural LLM / RAG (§2.3) | partial | – | ✓ | – | – |
| Selective prediction (§2.4) | ✓ | – | – | ✓ | – |
| **AI Plant Doctor (ours)** | ✓ (calibrated, OOD) | ✓ | ✓ (cited, IPM) | ✓ (τ, follow-up, expert) | ✓ |

To our knowledge, no prior system combines all five properties for tomato growers, and none ties them to the reasoning steps of a plant doctor.

## New references to add

- S. P. Mohanty, D. P. Hughes, M. Salathé. Using deep learning for image-based plant disease detection. *Frontiers in Plant Science*, 2016.
- K. P. Ferentinos. Deep learning models for plant disease detection and diagnosis. *Computers and Electronics in Agriculture*, 2018.
- B. Silva et al. GPT-4 as an agronomist assistant? Answering agriculture exams using large language models. arXiv, 2023.
- P. Lewis et al. Retrieval-augmented generation for knowledge-intensive NLP tasks. *NeurIPS*, 2020.
- S. Yao et al. ReAct: Synergizing reasoning and acting in language models. *ICLR*, 2023.
- C. Guo, G. Pleiss, Y. Sun, K. Q. Weinberger. On calibration of modern neural networks. *ICML*, 2017.
- W. Liu, X. Wang, J. Owens, Y. Li. Energy-based out-of-distribution detection. *NeurIPS*, 2020.
- Y. Geifman, R. El-Yaniv. Selective classification for deep neural networks. *NeurIPS*, 2017.
- H. Mozannar, D. Sontag. Consistent estimators for learning to defer to an expert. *ICML*, 2020.
