# Sensor-Driven 3D Digital Twin with Live State Synchronization

> **Status:** Active Research Project (Work in Progress)

An interactive **3D Digital Twin** of a CNC milling machine that continuously synchronizes with sensor data while visualizing **machine health** through an integrated predictive maintenance model.

The project combines **Industrial IoT**, **Machine Learning**, and **3D Visualization** into a lightweight digital twin capable of replaying real industrial sensor streams and displaying live machine state inside a browser-based 3D environment.

---

# Overview

A digital twin is more than a 3D model, it is a virtual representation that remains synchronized with a physical asset and supports intelligent decision-making.

This project implements both essential characteristics of a modern industrial digital twin:

- **Timeline-based state synchronization** — a JS playback loop advances an index into a down-sampled sensor timeline every frame and maps each channel to a visual property (see Method Overview)
- **Predictive maintenance using machine learning** — a Gradient Boosting Classifier trained on 5 sensor features, scoring every row with a failure probability that drives a warning beacon on the twin

To remain lightweight and reproducible, the prototype uses the **AI4I 2020 Predictive Maintenance Dataset** (10,000 real sensor readings from a milling process) as a replayable sensor stream while maintaining the same architecture that would be used for live IoT data.

---

# Pipeline

<p align="center">
    <img src="pipeline_diagram.svg" width="900">
</p>

<p align="center">
<b>Figure.</b> End-to-end digital twin architecture integrating industrial sensor data, predictive maintenance, and browser-based 3D visualization.
</p>

---

# Features

## Features

- ✅ Browser-based 3D digital twin (Three.js, self-contained HTML — no server)
- ✅ Procedural CNC milling machine model (base, column, head, table, spindle, tool bit)
- ✅ Real 10,000-row AI4I 2020 sensor dataset (not simulated)
- ✅ Timeline-replay synchronization at 1×/4×/16× playback speed
- ✅ Gradient Boosting failure-probability model driving a live health beacon
- ✅ Pause / seek / scrub playback controls
- ✅ Offline self-contained HTML export (`digital_twin.html`)
- ✅ Per-channel value-range normalization for color/gauge mapping

## Planned

- 📌 ESP32 / Arduino live sensor stream over WebSocket/MQTT
- 📌 Photorealistic machine reconstruction (photogrammetry)
- 📌 Gaussian Splatting–based geometry (reusing the companion SDS/DiT pipeline)
- 📌 Multiclass failure-mode prediction (tool wear / heat / power / overstrain / random)
- 📌 Time-to-failure (RUL) forecasting model
- 📌 Cloud deployment of the live-sync backend

---

# Method Overview

The digital twin follows a four-stage processing pipeline, with the specific
implementation detail at each step below.

```
AI4I 2020 sensor CSV — 10,000 rows × 5 features + failure label
          │  (HF mirror → UCI CSV → synthetic fallback chain)
          ▼
Column normalization → [air_temp, process_temp, rpm, torque, tool_wear, failure]
          │
          ▼
Gradient Boosting Classifier (scikit-learn), stratified 70/30 train/test split
          │
          ▼
Per-row failure probability fail_prob ∈ [0,1], scored for all 10,000 rows
and appended as an extra column on the sensor timeline
          │
          ▼
Procedural Three.js machine — base, column, head, table, spindle, tool bit,
warning beacon — plus the down-sampled timeline (every 10th row) and
per-channel min/max ranges, all embedded into one HTML file
          │
          ▼
JS playback loop: advance timeline index → look up row → normalize each
channel against its dataset range → map to a visual property (below)
          │
          ▼
Self-contained digital_twin.html — pause / seek / 1×-4×-16× speed controls
```

The workflow consists of:

1. Loading the AI4I 2020 CSV and normalizing column names.
2. Training a Gradient Boosting Classifier on 5 sensor features to predict `failure`.
3. Scoring every row with `fail_prob`, added to the timeline as an extra channel.
4. Embedding geometry + timeline + value ranges into one self-contained HTML file.
5. Running a per-frame JS loop that maps each sensor channel to a specific visual property (see table in "Live Synchronization" below).

The architecture is designed so that replacing the replay dataset with a live IoT stream requires only swapping the "advance timeline index" step for "receive new reading from socket" — the prediction and rendering logic don't change.

---

# Planned Methodology: World-Model-Style Interactive Twin (Phase 5)

**Status: Not implemented — conceptual design for a Phase 5 roadmap item.**
This section describes *how* the interactive world-model extension would
work if built, following the same three-component structure used by
world models like Genie, adapted from pixel/video space to this project's
sensor-state space.

## Motivation

The current twin is a **passive replay**: it plays back a fixed,
pre-recorded sensor timeline, and no input from the user changes what
happens next. A world-model-style twin would instead let a user (or a
control policy) **take an action** — e.g., "increase spindle RPM," "add
cutting load," "simulate a worn tool" — and have the twin **generate a
plausible resulting sensor trajectory**, rather than only replaying what
was recorded. This turns the twin from a visualization into a lightweight
**simulator** that could be used to explore hypothetical/what-if machine
states, including ones never observed in the AI4I dataset.

## Proposed architecture, mapped from Genie's three components

World models like Genie are built from three pieces: a tokenizer that
compresses raw video into discrete units, an autoregressive dynamics
model that predicts the next unit given past units, and a latent action
model that infers what "action" occurred between two consecutive frames
without needing action-labeled data. The table below maps each of those
onto an analog in this project's much lower-dimensional sensor-state
setting:

| Genie component | Role in Genie | Proposed analog for this twin |
|---|---|---|
| Video tokenizer | Compresses raw pixels into discrete visual tokens | Not needed as-is — sensor state is already a low-dimensional vector `[air_temp, process_temp, rpm, torque, tool_wear]`; this step is effectively free here |
| Autoregressive dynamics model | Predicts next video token(s) from past tokens | A small sequence model (e.g. GRU/LSTM or a tiny Transformer) trained to predict `state_{t+1}` from `state_t` + the chosen action, using the AI4I timeline as training sequences |
| Latent action model | Infers implicit actions between frames from unlabeled video | Not needed as unlabeled inference — **actions can be defined explicitly** for this domain (e.g. `Δrpm`, `Δtorque`, `advance_tool_wear`), since the sensor channels and their physical meaning are already known, unlike raw internet video |

This is a meaningfully **simpler** problem than Genie's: Genie has to
learn actions from unlabeled pixels because internet video has no action
log, and has to generate high-dimensional video frames. This project's
"frames" are 5 numbers, and the actions are known machine controls — so a
world-model-style dynamics model here is a small sequence-prediction
model, not a large-scale video generator.

## Planned input / output

| | Input | Output |
|---|---|---|
| Training the dynamics model | Sequences of `(state_t, action_t, state_{t+1})` derived from the AI4I timeline (actions inferred as the observed deltas between consecutive real readings, as a starting approximation) | A model predicting `state_{t+1}` given `state_t` and a chosen action |
| Interactive use (what the user experiences) | A discrete action choice at each step (e.g. "increase RPM," "simulate added load") | A generated next sensor state, fed into the *same* existing visual-mapping logic |

---

# Implementation Details

## Dataset

**AI4I 2020 Predictive Maintenance Dataset** — 10,000 rows, 5 real-valued sensor channels + 1 binary label:

| Raw column | Normalized name | Meaning |
|---|---|---|
| Air temperature [K] | `air_temp` | Air temperature (K) |
| Process temperature [K] | `process_temp` | Process temperature (K) |
| Rotational speed [rpm] | `rpm` | Spindle rotational speed |
| Torque [Nm] | `torque` | Torque (Nm) |
| Tool wear [min] | `tool_wear` | Tool wear (minutes) |
| Machine failure | `failure` | Binary failure label |

Dataset loading fallback chain (in order):
1. Hugging Face Hub mirrors (tried first; none of the attempted IDs resolved in testing)
2. UCI Machine Learning Repository CSV (canonical source — this is what the pipeline actually loaded in the verified run)
3. Synthetic fallback (development/offline use only, clearly labeled in logs when reached)

---

## Machine Learning

Current implementation:

- **Model:** `GradientBoostingClassifier` (scikit-learn, default hyperparameters, `random_state=0`)
- **Features:** `[air_temp, process_temp, rpm, torque, tool_wear]` (5 numeric columns)
- **Target:** `failure` (binary)
- **Split:** stratified 70/30 train/test (stratified because failures are a small minority class, ~3–4% of rows)
- **Output:** a failure probability `fail_prob ∈ [0,1]` computed for every row in the dataset, not just the test set — this is what drives the twin's warning beacon

Future versions will evaluate:

- XGBoost / LightGBM / Random Forest as alternative classifiers
- A deep learning model (small MLP or 1D-CNN over the sensor window) for comparison

---

## 3D Visualization

Built using **Three.js**, geometry generated procedurally (no imported mesh):

- Base
- Column
- Machine head
- Table
- Rotating spindle assembly
- Tool bit
- Status beacon

---

## Live Synchronization

Current synchronization mechanism: a `requestAnimationFrame` loop advances
a playback index through the down-sampled timeline (every 10th reading,
~1,000 points). Each frame, the current row's values are normalized
against that channel's dataset-wide min/max and mapped to a specific
visual property:

| Sensor | Visualization | Mapping detail |
|---------|---------------|---|
| `rpm` | Spindle rotation speed | angular velocity ∝ normalized rpm |
| `process_temp` | Machine body color | linear color interpolation, gray → red |
| `torque` | Head inclination | head Z-rotation ∝ normalized torque |
| `tool_wear` | Tool bit deformation | tool Y-scale shrinks, color darkens |
| `fail_prob` | Health beacon | 3-level discrete color: green / amber / red |

---

## Export

The project exports a single standalone file:

```
digital_twin.html
```

which contains:

- 3D geometry (Three.js scene graph, built in-script)
- The down-sampled sensor timeline (JSON, embedded inline)
- Per-channel value ranges (for normalization/color scales)
- The playback loop and UI controls

No backend server, database, or external file dependency is required —
opening the HTML file in any browser is sufficient.

---

# Training Configuration

| Setting | Value |
|----------|-------|
| Dataset | AI4I 2020 Predictive Maintenance (10,000 rows) |
| Features | `air_temp, process_temp, rpm, torque, tool_wear` |
| Target | `failure` (binary) |
| ML Library | scikit-learn |
| Classifier | `GradientBoostingClassifier` (default params) |
| Train/test split | stratified 70/30 |
| Visualization | Three.js r128 (CDN) |
| Language | Python (data + model) + JavaScript (twin) |
| Output | Standalone `digital_twin.html` |
| Platform | Kaggle (no GPU required) |

---

# Current Progress

The repository is actively evolving.

### Completed

- [x] AI4I 2020 dataset integration with HF → UCI → synthetic fallback chain
- [x] Column normalization to a standard schema
- [x] Gradient Boosting failure-probability model
- [x] Procedural Three.js machine construction
- [x] Timeline-replay synchronization with per-channel visual mapping
- [x] Interactive playback controls (pause / seek / speed)
- [x] Self-contained HTML export

### Currently Working On

- [ ] Live data synchronization (WebSocket/MQTT bridge)
- [ ] Improved predictive accuracy (alternative classifiers)
- [ ] Higher-fidelity machine animation
- [ ] Dashboard/HUD improvements
- [ ] Code cleanup and modularization

---

# Roadmap

## Phase 1 — Digital Twin Foundation

- [x] Sensor dataset (AI4I 2020, real data)
- [x] Gradient Boosting predictive model
- [x] Three.js procedural visualization
- [x] Timeline playback
- [x] Self-contained HTML export

---

## Phase 2 — Live Synchronization

- [ ] WebSocket integration
- [ ] MQTT communication
- [ ] ESP32 data streaming
- [ ] Real-time (non-replay) synchronization

---

## Phase 3 — Machine Intelligence

- [ ] Multiclass failure-mode classification (5 AI4I failure types)
- [ ] Time-to-failure prediction
- [ ] Remaining Useful Life (RUL) estimation
- [ ] Prediction confidence/uncertainty estimation

---

## Phase 4 — Visualization Improvements

- [ ] Photorealistic machine model (photogrammetry or Gaussian Splatting)
- [ ] Expanded dashboard with historical trend charts
- [ ] Interactive analytics panel

---

## Phase 5 — Research Extensions

- [ ] Gaussian Splatting reconstruction (reusing the companion 3D generation repo)
- [ ] Photogrammetry integration
- [ ] Cloud-hosted live digital twin
- [ ] Multi-machine monitoring
- [ ] Comparative ML benchmarking (GBM vs. XGBoost vs. deep learning)
- [ ] Industrial IoT deployment
- [ ] Diffusion-generated twin geometry (replacing procedural Three.js model with a video/3D-diffusion-reconstructed one)
- [ ] World-model-style interactive simulation (action-controllable environment, not just passive replay)

### Recent related work (2025–2026) informing this roadmap

- **Digital Twin Generation from Visual Data: A Survey** (2025) — surveys
  generating 3D digital twins directly from visual data using 3D Gaussian
  Splatting, generative inpainting, and diffusion-based foundation models;
  directly relevant to the planned Gaussian Splatting/photogrammetry items
  above.
- **Pantheon360** (CVPR 2026) — a 3D-aware 360° video diffusion model that
  generates high-fidelity, geometrically consistent scenes for downstream
  simulation and digital-twin applications, using an explicit 3D "cache"
  as a geometric scaffold so the diffusion model only needs to handle
  texture refinement. A candidate approach for the "diffusion-generated
  twin geometry" item above.
- **World models (Genie, Genie 2, NVIDIA Cosmos World Foundation Model
  Platform)** — extend generative simulation into embodied,
  action-controllable 3D environments rather than passive video/replay.
  Conceptually the most ambitious direction for this repo: today's twin
  *replays* a fixed sensor timeline, whereas a world-model-style twin
  would let a user or controller take actions and have the environment
  respond generatively — closer to a true simulator than a visualization.
  This is a longer-term research direction, not a near-term implementation
  target.


---

# Results

🚧 **Experimental results will be added as the project progresses.**
No metrics are reported yet — the values below are the categories that
will be filled in once the current training run is verified, not
placeholders for numbers already claimed:

- ROC-AUC, Precision, Recall, F1-score, Confusion Matrix (predictive model)
- Prediction latency
- Browser rendering frame rate
- Synchronization latency (replay vs. eventual live feed)

Visualization results will include:

- Live synchronization demo (recording/GIF)
- Interactive dashboard screenshots
- Failure-prediction walkthrough examples

---

# References

1. **Matzka, S.**
   *Explainable Artificial Intelligence for Predictive Maintenance Applications.*
   2020.

2. **AI4I 2020 Predictive Maintenance Dataset**
   UCI Machine Learning Repository.

3. **Grieves, M.**
   *Digital Twin: Manufacturing Excellence through Virtual Factory Replication.*
   2014.

4. **Tao, F., et al.**
   *Digital Twins and Cyber–Physical Systems toward Smart Manufacturing and Industry 4.0.*
   Engineering, 2019.

5. **Three.js Documentation**
   https://threejs.org/

6. **Digital Twin Generation from Visual Data: A Survey.**
   arXiv, 2025.

7. **Pantheon360: Taming Digital Twin Generation via 3D-Aware 360°
   Video Diffusion.**
   CVPR, 2026.

8. **Genie 2: A Large-Scale Foundation World Model.**
   DeepMind, 2024.

9. **NVIDIA Cosmos World Foundation Model Platform for Physical AI.**
   NVIDIA, 2025.

---

# Acknowledgements

This repository is an ongoing research project exploring the integration of **Industrial IoT**, **Machine Learning**, and **interactive 3D visualization** for predictive maintenance and digital twin applications. The implementation is designed to be lightweight, reproducible, and easily extensible toward real-world industrial deployments.

---

## Project Status

> 🚧 **Active Research Repository**
>
> Development is ongoing. Features, APIs, and visualization components may change as new experiments, live synchronization capabilities, and predictive models are integrated.
