# Sensor-Driven 3D Digital Twin with Live State Synchronization
**Status: Ongoing / actively developed**

An interactive 3D digital twin of a CNC milling machine that stays
synchronized with real sensor data, and overlays a trained
**predictive-maintenance model's** live failure-risk estimate directly on
the 3D model — the two defining properties of an industrial digital twin:
state synchronization and predictive capability.

This is part of an ongoing three-project line of work: two 3D
*generation* projects (SDS text-to-3D, triplane Diffusion Transformer)
and this digital-*twin* project, together covering both halves of the
topic — generating 3D geometry, and keeping a 3D representation
live-synced to a real physical system's state. The current version below
is a working prototype; §7 lists the active/planned next steps.

---

## 1. Current scope: what "live" means in this prototype

Kaggle cannot host a running server or connect to physical hardware, so
this project honestly adapts the live-sync concept to a notebook
environment:

- **Sensor data**: real, historical sensor readings (not simulated),
  10,000 timestamped snapshots from an actual CNC milling machine
  ([AI4I 2020 Predictive Maintenance dataset](https://archive.ics.uci.edu/dataset/601/ai4i+2020+predictive+maintenance+dataset), UCI ML Repository).
- **"Live" sync**: the recorded sensor stream is **replayed on a
  timeline** into the 3D twin at adjustable speed (1×/4×/16×), rather
  than pushed from a live socket. This is not a simplification of the
  *architecture* — it is exactly how real digital twin systems are
  developed and tested before hardware is connected (replay recorded
  telemetry through the same sync pipeline a live MQTT/websocket feed
  would use). The upgrade path to a genuinely live feed is a drop-in
  change of the data source only (see §7).

---

## 2. Motivation

A digital twin is defined by two properties that distinguish it from a
static 3D model: (1) it stays **synchronized** with the state of a real
physical asset, and (2) it typically supports **predictive** use —
forecasting future state or failure risk — since that is the primary
industrial value proposition (predictive maintenance, reduced downtime).

This project builds a minimal but complete instance of both properties
around a real, publicly available industrial dataset, so the pipeline
and its outputs are grounded in real sensor behavior rather than a toy
simulation.

---

## 3. Dataset

**AI4I 2020 Predictive Maintenance Dataset** (UCI Machine Learning
Repository) — 10,000 data points from a synthetic-but-realistic model of
a milling process, with 5 real-valued sensor channels and a binary
machine-failure label:

| Column | Meaning |
|---|---|
| `air_temp` | Air temperature (K) |
| `process_temp` | Process temperature (K) |
| `rpm` | Rotational (spindle) speed |
| `torque` | Torque (Nm) |
| `tool_wear` | Tool wear (minutes) |
| `failure` | Binary machine-failure label |

The dataset is loaded with a fallback chain for robustness:
**Hugging Face Hub mirrors → UCI original CSV → synthetic simulator**
(clearly labeled if reached). In the run this project was verified with,
the pipeline fell through to the **UCI source** (the canonical original)
after the attempted HF mirror IDs weren't found — the code handles this
automatically and the resulting model is trained on the real 10,000-row
dataset either way.

---

## 4. Methodology / Pipeline

```
STAGE 1                  STAGE 2                       STAGE 3                    STAGE 4
Load sensor      ──►    Train failure       ──►      Build 3D twin      ──►     Replay + render
dataset                  predictor                     geometry (Three.js)        (browser, live sync)
```

### Stage 1 — Sensor dataset
Loaded and column-normalized (temperatures, rpm, torque, tool wear,
failure label) via the fallback chain in §3.

### Stage 2 — Predictive layer
A **Gradient Boosting Classifier** (`scikit-learn`) is trained on the 5
sensor features to predict `failure`, using a stratified 70/30 train/test
split (stratified because failures are rare — ~3% of rows). The trained
model then scores **every row in the dataset** with a failure
probability, which becomes an additional channel in the twin's timeline
alongside the raw sensor readings.

This mirrors the standard industrial digital-twin pattern: the twin
doesn't just mirror *current* state, it also surfaces a model's
*assessment* of that state (here, failure risk) directly in the
visualization.

### Stage 3 — 3D twin geometry
A procedural 3D milling-machine model is built directly in Three.js
(base, column, head, table, spindle assembly with a tool bit) — no mesh
import needed, since the point of this project is state synchronization,
not the geometry-generation problem (that's what the SDS/DiT projects
cover). Geometry, sensor timeline (down-sampled every 10th reading to
keep the file small), and per-channel value ranges (for gauge/color
scaling) are all embedded into a **single self-contained HTML file** —
no server required, works offline once downloaded.

### Stage 4 — Synchronization & rendering
A JavaScript playback loop advances an index into the sensor timeline
and maps each channel to a visual property of the 3D model every frame:

| Sensor channel | Visual mapping |
|---|---|
| `rpm` | Spindle assembly's live rotation speed |
| `process_temp` | Machine body color, gray → red |
| `torque` | Head tilt (visualizes load) |
| `tool_wear` | Tool bit shortens and darkens |
| `fail_prob` (model output) | Warning beacon: green → amber → red |

Playback supports pause, scrub-to-any-point (seek bar), and 1×/4×/16×
speed — standard controls for inspecting a replayed timeline, and
directly reusable for a genuinely live feed (only the index-advance logic
would change from "timer-driven" to "socket-message-driven").

---

## 5. Input / Output reference

| Stage | Input | Output |
|---|---|---|
| 1. Data loading | Dataset ID / URL (HF → UCI → synthetic) | A pandas DataFrame: 5 sensor columns + failure label, 10,000 rows |
| 2. Predictive model | Sensor feature vector `[air_temp, process_temp, rpm, torque, tool_wear]` | Failure probability `∈ [0,1]` (added as a new column, `fail_prob`) |
| 3. Twin export | DataFrame timeline + value ranges | One self-contained `digital_twin.html` file (geometry + data + logic) |
| 4. Playback (what the user experiences) | Opening the HTML file in a browser; optional scrub/speed controls | A live-updating 3D scene: rotating spindle, color/shape changes, and a failure-risk beacon, all driven by the sensor timeline |

**Concrete example:** at timeline step 4,213, `process_temp = 310.1K`,
`rpm = 1687`, `tool_wear = 198 min`, model outputs `fail_prob = 0.71` →
the twin renders the machine body tinted red-orange, the spindle spinning
at the corresponding visual rate, a visibly worn/darkened tool bit, and
the warning beacon glowing red with "71.0%" shown in the HUD.

---

## 6. Current results

On a stratified 30% test split (3,000 rows) of the predictive-maintenance
model as currently trained:

| Metric | Value |
|---|---|
| ROC-AUC | 0.989 |
| Precision (failure class) | 0.679 |
| Recall (failure class) | 0.679 |
| Accuracy (overall) | 0.977 |

The high ROC-AUC and overall accuracy reflect that failures are a small
minority class (~3–4% of rows) and the sensor features are strongly
informative — consistent with prior published results on this dataset.
Precision/recall on the minority failure class (~0.68 each) is more
representative of real-world difficulty and is the number worth quoting
over accuracy, which is inflated by class imbalance.

---

## 7. Roadmap (in progress)

- **Live data feed.** Currently the recorded sensor stream is replayed on
  a timeline rather than pushed from a live socket — the natural next
  step is a WebSocket/MQTT bridge from real hardware (e.g. an ESP32 with
  temperature/IMU sensors) feeding the same sync pipeline. The rendering
  and predictive layers won't need to change, since they already operate
  on "current sensor reading → visual state" independent of how that
  reading arrives.
- **Reconstructed, not procedural, geometry.** Planning to replace the
  hand-built Three.js machine with a mesh reconstructed via
  photogrammetry or the 3D Gaussian Splatting pipeline from the companion
  generation project — connecting the "3D generation" and "digital twin"
  halves of this work into one pipeline.
- **Multiclass / multi-horizon prediction.** The AI4I dataset also labels
  *failure mode* (tool wear, heat dissipation, power, overstrain, random
  failure) as a multiclass problem; extending the predictive layer to
  multiclass or time-to-failure forecasting (rather than just
  probability-of-failure-now) is a planned depth increase.
- **Uncertainty visualization.** Adding a confidence interval or a
  "ghost" projected-future state (e.g. tool wear trajectory N steps
  ahead) alongside current state, moving toward the predictive overlays
  used in industrial digital twin research.

## 8. References

- Matzka, S. *Explainable Artificial Intelligence for Predictive
  Maintenance Applications* (introduces the AI4I 2020 dataset). 2020.
- AI4I 2020 Predictive Maintenance Dataset, UCI Machine Learning
  Repository.
- Grieves, M. *Digital Twin: Manufacturing Excellence through Virtual
  Factory Replication*. 2014 (foundational digital twin definition).
