# SGSMA 2026 — Synchrophasor Anomaly Detection Competition

**Anomaly / Event Detection, Classification, and Localization using PMU Data on the IEEE 39-Bus Test System**

Part of the **5th International Conference on Smart Grid Synchronized Measurements & Analytics (SGSMA 2026)**.

![Status](https://img.shields.io/badge/status-open-brightgreen)
![Data](https://img.shields.io/badge/data-PMU%20time--series-blue)
![System](https://img.shields.io/badge/test%20system-IEEE%2039--bus-orange)
![Sampling](https://img.shields.io/badge/rate-30%20fps-lightgrey)

---

## Table of Contents

- [Overview](#overview)
- [Key Dates](#key-dates)
- [Competition Goal](#competition-goal)
- [Quick-Start Checklist](#quick-start-checklist)
- [Repository Contents](#repository-contents)
- [The Power System Model](#the-power-system-model)
- [PMU-to-Bus Mapping](#pmu-to-bus-mapping)
- [Event Timeline (Ground Truth)](#event-timeline-ground-truth)
- [Dataset Format](#dataset-format)
- [Event Labels](#event-labels)
- [Missing Data](#missing-data)
- [The Three Tasks](#the-three-tasks)
- [Modelling Guidance](#modelling-guidance)
- [Data Splitting & Leakage Prevention](#data-splitting--leakage-prevention)
- [Evaluation Metrics](#evaluation-metrics)
- [Efficiency-Aware Scoring](#efficiency-aware-scoring)
- [Required Deliverables](#required-deliverables)
- [Resources](#resources)
- [Ethics & Academic Integrity](#ethics--academic-integrity)
- [FAQ](#faq)
- [Organizers & Contact](#organizers--contact)

---

## Overview

Phasor Measurement Units (PMUs) are high-precision grid sensors that report **time-synchronized** voltage and current phasors at **30–60 frames per second**, all referenced to a common GPS clock with sub-microsecond accuracy. Unlike traditional SCADA (which reports every 2–10 seconds and without precise time synchronization), PMUs can observe fast transients and inter-area oscillations as they happen.

This competition challenges participants to build an algorithm that ingests **multivariate PMU time-series data** and, in real-time fashion, **detects**, **classifies**, and **localizes** disturbances on the grid — while keeping the model **accurate and efficient**.

The dataset was produced from a **dynamic time-domain simulation of the IEEE 39-bus (New England) test system**, with a carefully orchestrated sequence of physical and cyber events injected over a ~90-minute window observed by **8 PMUs**.

> This README summarizes the official **Competition Guide** (`SGSMA 2026_Anomaly Detection Competition User Guide.pdf`, Rev. February 18, 2026). If anything here conflicts with that document, **the official guide takes precedence.**

---

## Key Dates

| Milestone | Date |
| --- | --- |
| Competition period | **February 15, 2026 – April 15, 2026** |
| Announcement of Top-3 teams | **April 25, 2026** |
| On-site final (Top-3 teams) | **June 01, 2026** — 90-minute session |

---

## Competition Goal

Design an algorithm that uses PMU time-series data to:

1. **Detect** abnormal behavior (normal vs. abnormal).
2. **Classify** the event type (fault, outage, generation trip, …).
3. **Localize** the event origin (bus and/or line).

…while balancing **accuracy** and **model efficiency** (fewer parameters earns a bonus).

---

## Quick-Start Checklist

- **Input data:** 8 CSV files (one per PMU bus), timestamps aligned across all buses, ~162,000 rows each.
- **Your output:** Code with predicted event labels & locations, appended to the files or in a separate submission CSV.
- **Report:** A 3–6 page PDF with metrics, confusion matrix, parameter count, runtime, and training/validation curves.
- **Data span:** ≈ 90 minutes at 30 frames/second (fps).

---

## Repository Contents

```
.
├── README.md                                  # This file
├── SGSMA 2026_Anomaly Detection Competition User Guide.pdf   # Official guide (authoritative)
├── Event Timeline & Location.xlsx             # Event schedule & ground-truth locations
├── IEEE 39 Bus Power System.raw               # PSS/E RAW network model (bus/branch/gen/load)
├── PMUbus_ Location.txt                        # Bus list with voltages, angles, PMU assignment
└── Training Dataset/                           # PMU measurement data (required inputs)
    ├── Bus2_Competition_Data_nanmask.csv       # PMU 6
    ├── Bus5_Competition_Data_nanmask.csv       # PMU 7
    ├── Bus6_Competition_Data_nanmask.csv       # PMU 8
    ├── Bus10_Competition_Data_nanmask.csv      # PMU 3
    ├── Bus19_Competition_Data_nanmask.csv      # PMU 5
    ├── Bus22_Competition_Data_nanmask.csv      # PMU 4
    ├── Bus29_Competition_Data_nanmask.csv      # PMU 2
    └── Bus39_Competition_Data_nanmask.csv      # PMU 1
```

**Required files:** the 8 PMU CSVs in `Training Dataset/`.
**Supplementary files (for topology-aware / graph models):** the `.raw` model, the PMU location text file, and the event-timeline spreadsheet.

> **Tooling note for the `.raw` file:** PSS/E RAW requires a commercial PSS/E licence. Free alternatives that can parse it include **pandapower** (Python), **MATPOWER** (MATLAB/Octave), **PYPOWER**, and **GridCal**.

---

## The Power System Model

The data comes from the **IEEE 39-bus test system** (the 10-machine New England system), one of the most widely used benchmarks in power-systems research.

| Component | Count | | Parameter | Value |
| --- | --- | --- | --- | --- |
| Buses | 39 | | MVA base | 100 MVA |
| Generators | 10 | | Rated frequency | 60 Hz |
| Transmission lines | 34 | | HV voltage | 345 kV |
| Transformers | 12 | | Generator voltage | 13.8 kV |

Only **8 of the 39 buses are monitored by PMUs**. When an event occurs at a non-PMU bus, your algorithm must infer its presence and location from indirect signatures observed at the 8 monitored buses. Events closer to a PMU produce stronger, faster signatures; distant events produce weaker, more diffuse responses — which is why **multi-PMU fusion** and **network topology** are critical, especially for localization.

---

## PMU-to-Bus Mapping

| PMU | Bus | kV | \|V\| (p.u.) | θ (deg) | CSV Filename |
| --- | --- | --- | --- | --- | --- |
| PMU 1 | 39 | 345.0 | 1.0300 | −10.05 | `Bus39_Competition_Data_nanmask.csv` |
| PMU 2 | 29 | 345.0 | 1.0499 | +0.75 | `Bus29_Competition_Data_nanmask.csv` |
| PMU 3 | 10 | 345.0 | 1.0172 | −5.43 | `Bus10_Competition_Data_nanmask.csv` |
| PMU 4 | 22 | 345.0 | 1.0498 | +0.67 | `Bus22_Competition_Data_nanmask.csv` |
| PMU 5 | 19 | 345.0 | 1.0499 | −1.02 | `Bus19_Competition_Data_nanmask.csv` |
| PMU 6 | 2  | 345.0 | 1.0487 | −5.75 | `Bus2_Competition_Data_nanmask.csv`  |
| PMU 7 | 5  | 345.0 | 1.0053 | −8.61 | `Bus5_Competition_Data_nanmask.csv`  |
| PMU 8 | 6  | 345.0 | 1.0077 | −7.95 | `Bus6_Competition_Data_nanmask.csv`  |

(Full 39-bus steady-state data is in the official guide and the supplementary files.)

---

## Event Timeline (Ground Truth)

A sequence of physical and cyber events is injected across the 90-minute window. Ground-truth locations are provided so participants understand the data structure; the per-sample `Event` column in each CSV encodes these events at every time step.

| # | Category | Event Description | Approx. Time | Location |
| --- | --- | --- | --- | --- |
| 1 | Cyber | Data drop at Bus 29 (PMU link failure — frames become NaN) | 10th min | Bus 29 |
| 3 | Physical | Three-phase line-to-ground (3LG) fault at Bus 39 | 20th min | Bus 39 |
| 4 | Physical | Line outage between Bus 24 and Bus 23 | 40th min | Bus 24 |
| 5 | Cyber | Data drop at Bus 29 (second communication failure) | 45th min | Bus 29 |
| 7 | Cyber + Physical | Concurrent: data drop at Bus 29 **and** generation change at Bus 2 | 50th min | Bus 29 & Bus 2 |
| 8 | Physical | Generation change at Bus 2 | 50th min | Bus 2 |
| 9 | Physical | Generation change at Bus 2 | 55th min | Bus 2 |
| 10 | Physical | Load change at Bus 7 | 65th min | Bus 7 |
| 11 | Physical | Load change at Bus 7 | 70th min | Bus 7 |

**Notes:**
- **Cyber events** affect only the targeted PMU's stream (NaN measurements, `DATA_PRESENT = 0`); other PMUs keep reporting.
- **Physical events** produce electromechanical transients that propagate across the network and appear at multiple PMUs with varying magnitude.
- **Event #7 is concurrent** — Bus 29's data is missing while a generation change happens at Bus 2, so detection must rely on the remaining PMUs.
- Scenario numbers are **non-consecutive** (#2 and #6 are absent); this does not affect the data. The CSV `Event` labels are independent of these scenario numbers.

---

## Dataset Format

Every CSV has the **same 17-column structure**. Each row is one PMU measurement frame at a specific timestamp (~30 fps).

| Column(s) | Description | # |
| --- | --- | --- |
| `TIMESTAMP` | Elapsed seconds from simulation start, rounded to 3 decimals (~30 fps spacing). | 1 |
| `VA_mag`, `VA_ang`, `VB_mag`, `VB_ang`, `VC_mag`, `VC_ang` | Three-phase **voltage** phasors: magnitude (V) and angle (deg) for phases A, B, C. | 6 |
| `IA_mag`, `IA_ang`, `IB_mag`, `IB_ang`, `IC_mag`, `IC_ang` | Three-phase **current** phasors: magnitude (A) and angle (deg) for phases A, B, C. | 6 |
| `Frequency` | System frequency in Hz (nominal 60 Hz). Deviations indicate generation/load imbalance. | 1 |
| `ROCOF` | Rate of change of frequency (Hz/s) — one of the fastest disturbance indicators. | 1 |
| `DATA_PRESENT` | `1` = valid frame; `0` = missing frame (all measurements NaN). | 1 |
| `Event` | Ground-truth label (integer 0–8). May be blank/withheld in the hidden test set. | 1 |

**Physical meaning of the features:**
- **Voltage magnitude** — bus voltage level; faults cause sharp sags (20–80%), load changes drift gradually.
- **Voltage angle** — phase position vs. GPS time; angle differences reveal power-flow direction; outages shift angles as power reroutes.
- **Current magnitude** — fault currents can spike to many times normal.
- **Current angle** — varies with power factor and fault characteristics.
- **Frequency** — global generation–load balance; loss of generation drops it, loss of load raises it (normal < 0.02 Hz; events 0.1–1.0 Hz).
- **ROCOF** — derivative of frequency; reacts fastest and may precede observable voltage/current changes by hundreds of milliseconds.

**Combining PMUs:** because timestamps are aligned across all files, a simple `pandas.merge()` on `TIMESTAMP` yields a combined matrix of **8 × 14 = 112 PMU features plus 8 `DATA_PRESENT` flags**. Multi-PMU models exploit spatial correlations and differential responses for better classification and localization.

---

## Event Labels

The `Event` column uses integer labels **0–8** (one integer per row, per PMU). Concurrent events use label 6 or 8.

| ID | Name | Description |
| --- | --- | --- |
| 0 | Normal operation | Steady or near-steady state; minor natural fluctuations may be present. |
| 1 | Fault | Short circuit (e.g., 3-phase-to-ground). Sudden voltage sags and current spikes lasting cycles to seconds. |
| 2 | Line outage | A transmission line is disconnected. Causes power-flow redistribution and voltage/angle shifts. |
| 3 | Generation change/outage | A generator changes MW output or trips. Causes frequency deviation and system-wide redistribution. |
| 4 | Load change/drop | A load suddenly increases, decreases, or disconnects. Similar to generation change but smaller frequency excursions. |
| 5 | Missing data | PMU frame missing (comm failure). All measurements NaN, `DATA_PRESENT = 0`. No physical event. |
| 6 | Missing data + physical event | Missing data at one PMU concurrent with a physical event elsewhere — infer it from the other PMUs. |
| 7 | Bad data | Corrupted frames: non-physical spikes/jumps; the PMU reports values but they are unreliable. |
| 8 | Unknown event | Open-set class for abnormal patterns that don't match labels 1–7. |

---

## Missing Data

When `Event = 5` (missing data only) or `Event = 6` (missing data concurrent with a physical event):
- All **14 PMU measurement columns become NaN** (voltages, currents, frequency, ROCOF).
- `DATA_PRESENT = 0`.

This simulates a real PMU communication failure — the device works, but its stream is interrupted.

**Strategies (document your choice in the report):**
1. **Masking** — keep NaNs and use `DATA_PRESENT` as a binary mask (masked Transformers, attention with padding masks, GNNs with missing-node indicators).
2. **Imputation** — forward-fill/LOCF, linear/spline interpolation, or a learned imputer (e.g., VAE). **Always keep `DATA_PRESENT` as an extra feature** so the model knows which values are synthetic.
3. **Hybrid** — imputation for classical ML pipelines, masking for deep sequence models.

> Missing-data periods are **not random** — they correspond to specific cyber events in the timeline. During Event 6, Bus 29's stream is unavailable while a generation change happens at Bus 2; your algorithm must detect the physical event from the remaining 7 PMUs.

---

## The Three Tasks

### Task 1 — Event Detection (binary)
Decide **normal (label 0)** vs. **abnormal (labels 1–8)**.
Challenges: low false-alarm rate, small detection delay, dominant normal class (threshold tuning).
Metrics: Precision, Recall, F1 (Abnormal), false alarm rate (FP/min), detection delay (s).

### Task 2 — Event Classification (multi-class)
Predict the specific event type (labels 1–8) once abnormal.
Challenges: class imbalance, overlapping signatures (generation vs. load changes), separating label 5 from label 6.
Metrics: Macro-F1, Weighted-F1, per-class P/R/F1, full confusion matrix.

### Task 3 — Event Localization
Estimate the **bus and/or line** of origin.
Challenges: non-PMU-bus events inferred from indirect signatures; multi-PMU + topology models outperform single-PMU; repeated events at the same bus (Bus 2 at min 50 & 55; Bus 7 at min 65 & 70).
Metrics: Top-1 accuracy, Top-3 accuracy, mean electrical-distance error (if provided).

> **Detection and Classification are required. Localization is strongly encouraged and earns additional credit.**

---

## Modelling Guidance

- **Windowing:** sliding windows of 1–5 s (30–150 samples). Shorter = faster detection; longer = captures slow transients/oscillations.
- **Normalization:** normalize with training-set statistics; robust scaling (median + IQR) preferred due to fault outliers.
- **NaN handling:** handle missing data explicitly (see above).
- **Multi-PMU fusion:** feature concatenation (112 features), attention-based fusion, or a graph neural network over the 39-bus topology (PMU buses = observed nodes, line impedances = edge weights).
- **Model families:** Temporal CNNs (1D conv), LSTM/GRU, Transformer encoders (with positional encoding), gradient-boosted trees (XGBoost/LightGBM) with hand-crafted window features, autoencoders (unsupervised scoring), hybrid pipelines (autoencoder → classifier).

---

## Data Splitting & Leakage Prevention

PMU time series are **highly autocorrelated**. Randomly shuffling rows then splitting causes severe **data leakage** — the same event window appears in both train and test, inflating performance by 20–40%.

> ⚠️ **Do NOT randomly shuffle time-series rows.** Always use **contiguous time-block splits.**

**Recommended split** (unless organizers provide a fixed one):
- **70%** training (earliest portion),
- **15%** validation (middle),
- **15%** internal testing (latest).

Ensure no sliding window straddles a split boundary; leave a buffer of ~one window length between splits.

---

## Evaluation Metrics

**Task 1 (Detection):** Precision/Recall/F1 for Abnormal; false alarm rate (FP per minute of normal operation); detection delay (s from onset to first correct Abnormal prediction).

**Task 2 (Classification):** Macro-F1, Weighted-F1, per-class Precision/Recall/F1, and a full **9 × 9 confusion matrix** (labels 0–8).

**Task 3 (Localization):** Top-1 accuracy, Top-3 accuracy, mean/median electrical-distance error (if a distance matrix is provided).

**Efficiency / complexity (required from every team):** trainable parameter count; model size on disk (MB) or equivalent description for non-neural methods; inference time (seconds per minute of PMU data) plus hardware used (CPU/GPU/RAM); window length.

---

## Efficiency-Aware Scoring

The evaluator team may use a composite score that rewards compact models:

```
Score = Macro-F1 − λ · log10(Number of Parameters)
```

Default **λ = 0.03**; sensible range **λ ∈ [0.02, 0.05]**.

*Example (λ = 0.03):* a 100K-parameter model pays `0.03 × 5 = 0.15`; a 10M-parameter model pays `0.03 × 7 = 0.21`. The larger model must score at least **0.06** higher Macro-F1 to compensate — encouraging both high accuracy and simplicity.

---

## Required Deliverables

### (A) Executable code with prediction files
Submit code (zipped) that runs on a standard machine. Predicted labels in one of:
1. **Inline:** add `Predicted_Event` and (optionally) `Predicted_Location` columns to each bus CSV, aligned by `TIMESTAMP`.
2. **Separate file(s):** CSV(s) with `TIMESTAMP, Bus, Predicted_Event, Predicted_Location`.

> ⚠️ **Timestamp alignment is critical.** Misaligned timestamps (rounding/order) produce scoring errors or zero credit.

### (B) Technical report (PDF, 3–6 pages)
At minimum: method overview (architecture, window length, features, fusion strategy); preprocessing (NaN handling, normalization, feature engineering); training details (optimizer, LR, batch size, epochs, augmentation, seeds); results (detection, classification, localization metrics + full confusion matrix); efficiency (params, size, inference time, hardware); training curves (loss and primary metric for train & validation — no test-set curves unless labels are provided); discussion (design decisions, failure modes, improvements).

---

## Resources

**IEEE 39-bus model sources**
- Texas A&M Electric Grid Test Cases (New England 39-bus): `electricgrids.engr.tamu.edu`
- PSS/E RAW downloads: `3phaseee.com/raw_files`
- MATPOWER: `matpower.org` · `case39` reference and source on the MATPOWER GitHub.

**Suggested Python ecosystem**
- **Data:** pandas, numpy, scipy
- **ML:** scikit-learn, xgboost, lightgbm
- **Deep learning:** PyTorch or TensorFlow/Keras
- **Time-series:** tslearn, tsfresh, sktime
- **Power systems:** pandapower (parse `.raw`, compute electrical distances)
- **Visualization:** matplotlib, seaborn, plotly

Any language is acceptable; Python is recommended for the widest library support.

---

## Ethics & Academic Integrity

- Use the dataset only for the intended competition purposes.
- Do not attempt to infer or reverse-engineer hidden-test-set labels.
- Follow academic-integrity standards, including attribution for external libraries, pre-trained models, and prior work.
- Ensure reproducibility: report seeds, key hyperparameters, library versions, and enough detail for organizers to verify.
- Do not share competition data or solutions outside your team during the competition period.

---

## FAQ

1. **Pre-trained models?** Allowed, but disclose all pre-trained components and count their parameters in the efficiency metric.
2. **External datasets for pre-training?** Public IEEE 39-bus simulation data is generally fine; proprietary utility data is not. When in doubt, ask.
3. **Which language?** Any (Python, MATLAB, Julia, R, …); Python recommended.
4. **Must I build all three tasks?** Detection and Classification are **required**; Localization is strongly encouraged (extra credit).
5. **How is the winner determined?** The committee scores detection accuracy, classification (macro-F1), localization, model efficiency, and report quality; exact weights announced by organizers.
6. **If my model can't predict a class?** You may assign label 8 (unknown) when uncertain — but overusing it hurts your macro-F1.
7. **Ensembles?** Allowed; report total parameter count and inference time of the full ensemble.
8. **How to compute electrical distance?** Parse the `.raw` with pandapower/MATPOWER to get Y<sub>bus</sub>, invert to Z<sub>bus</sub>; the electrical distance between buses *i* and *j* is `|Z_ii + Z_jj − 2·Z_ij|`.

---

## Organizers & Contact

| Organizer | Affiliation | Email |
| --- | --- | --- |
| **Masoud Barati** | University of Pittsburgh | masoud.barati@pitt.edu |
| **Payman Dehghanian** | George Washington University | payman@gwu.edu |
| **David Celeita** | Universidad de la Sabana | david.celeita@unisabana.edu.co |

**Good luck and have fun!**

---

<sub>This README summarizes the official Competition Guide (Rev. February 18, 2026). In any case of conflict, the official PDF guide is authoritative.</sub>
