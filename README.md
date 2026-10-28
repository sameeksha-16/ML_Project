# ML_Project


# CAN Network Intrusion Detection Using Machine Learning

Detecting and classifying attacks on the in-vehicle **Controller Area Network (CAN) bus** with machine learning, using the public CAN Intrusion Dataset from IEEE DataPort (Mercedes ML350).

**Team:** Prateek Vamana Pai (240905006) · Dhrithi D Shetty (240905004) · Sameeksha Bajoria (240905190)

**Keywords:** Controller Area Network (CAN), Intrusion Detection, Automotive Cybersecurity, Machine Learning, Anomaly Detection

---

## Table of contents

1. [About the project](#about-the-project)
2. [Dataset](#dataset)
3. [What we did](#what-we-did)
4. [Features](#features)
5. [Experimental setup](#experimental-setup)
6. [Results and figures](#results-and-figures)
7. [Key findings](#key-findings)
8. [How the project meets its objectives](#how-the-project-meets-its-objectives)
9. [Project structure and how to run](#project-structure-and-how-to-run)
10. [Limitations](#limitations)
11. [Reference](#reference)

---

## About the project

### The problem

A modern car contains dozens of small computers called Electronic Control Units (ECUs) that control the engine, transmission, steering and more. They talk to each other over a shared network, the **CAN bus**. The original CAN protocol was designed with **no authentication, no authorization and no encryption**, so every ECU simply trusts every message. An attacker who gets onto the bus (for example through a compromised ECU or the diagnostic port) can inject fake messages and interfere with the vehicle.

An **intrusion detection system (IDS)** watches the bus traffic and raises an alarm when something looks wrong. This project builds and evaluates machine-learning-based IDS models on real CAN traffic.

### The attacks studied

| Attack | What the attacker does |
|---|---|
| **DoS (denial of service)** | Floods the bus with the highest-priority message ID (`0x000`), so legitimate messages are delayed or blocked |
| **Fuzzy (fuzzing)** | Injects messages with random payloads on existing IDs, to confuse ECUs or trigger faults |
| **Impersonation** | Pretends to be a real ECU by sending messages with its ID (`0x164`) and a fake payload |

### Objectives

1. Analyse and preprocess real-world CAN bus traffic containing normal and malicious communication.
2. Develop machine-learning models that **detect and classify** different types of CAN intrusions.
3. Compare the models using appropriate classification and intrusion-detection measures.

The project also looks at several challenges noted in the literature: class imbalance, informative feature selection, computational cost, unseen attacks, and generalisation across vehicles.

---

## Dataset

M. Sami, "Intrusion Detection in CAN bus," IEEE DataPort, 2019. doi: [10.21227/24m9-a446](https://doi.org/10.21227/24m9-a446)
Download: https://ieee-dataport.org/documents/can-intrusion-dataset

The data files are **not included** in this repository. Download them and place them in the project root.

| File | Messages | Duration | Unique IDs | Attack share (our labels) |
|---|---|---|---|---|
| `Attack_free_dataset.txt` | 2,369,398 | 1,037.6 s | 45 | 0 % |
| `DoS_attack_dataset.txt` | 656,579 | 306.3 s | 46 | 51.12 % |
| `Fuzzy_attack_dataset.txt` | 591,990 | 347.3 s | 45 | 36.23 % |
| `Impersonation_attack_dataset.txt` | 995,472 | 482.6 s | 45 | 0.57 % |

Each line is one CAN message:

```
Timestamp: 0.000271  ID: 0080  000  DLC: 8  00 17 dc 09 16 11 16 bb
```

Fields: timestamp (seconds), message ID (hex), a small extra field (not used), DLC (number of data bytes, 0 to 8) and the data bytes.

### Labelling (the files have no label column)

The files do not say which message is an attack, so we derived labels from how each attack looks:

| Attack | Rule used to label attack messages | Attack messages |
|---|---|---|
| DoS | ID `0x000` (flooding, all-zero payload) | 335,624 |
| Fuzzy | ID has at most 20 distinct payloads in normal data, **and** the (ID, payload) pair never occurs in attack-free data, **and** it occurs at most twice in the Fuzzy file | 214,483 |
| Impersonation | ID `0x164` with the constant payload `01 02 03 04 05 06 07 08` | 5,697 |

> **Important:** Fuzzy and Impersonation labels are heuristic. Results measure agreement with these rules, not official ground truth.

---

## What we did

The project is a pipeline of 16 Python scripts, each producing the input of the next:

1. **Parsed** the raw text files into tables (timestamp, ID, DLC, 8 data bytes).
2. **Explored** the traffic: message rates, ID frequencies, timing, and what each attack looks like.
3. **Labelled** attack messages with the rules above (Fuzzy also uses the attack-free file as a dictionary of normal behaviour).
4. **Engineered features** from the current and past messages: bus-wide timing, per-ID behaviour and payload statistics.
5. **Trained and compared four models** (Logistic Regression, Decision Tree, Random Forest, Gradient Boosting) separately for each attack, on three levels of information: full features, context features without raw ID/bytes, and timing-only.
6. **Built one four-class model** that classifies a message as Normal, DoS, Fuzzy or Impersonation, plus a check for whether a model could "cheat" by recognising the capture session.
7. **Tested anomaly detection** (Isolation Forest and autoencoder trained on normal traffic only) and **unseen-attack generalisation** (train without one attack, test on it).
8. **Studied feature selection and cost**: how few features are enough, and how fast inference is.
9. **Plotted** everything for the report.

---

## Features

All features are causal (they use only the current and past messages, as a real-time IDS must). Window = last 100 messages.

| Group | Features |
|---|---|
| Bus-wide timing | `dt_prev` (time since previous message), `win_mean_dt`, `win_std_dt` |
| Per-ID behaviour | `dt_same_id` (time since previous message with the same ID), `bytes_changed` and `bytes_changed_lag2` (bytes that differ from the previous / second previous message with the same ID), `id_count_win` (how often this ID appeared in the window), `dlc` |
| Payload statistics | payload entropy, number of distinct byte values, byte mean, byte standard deviation, number of zero bytes |
| Raw message | `can_id`, bytes `d0` to `d7` (used in the "Full" feature set only) |

---

## Experimental setup

- **Time-based split inside each file:** first 70 % of the capture for training, last 30 % for testing. (Neighbouring CAN messages are almost identical, so a random split would inflate scores.)
- **Imbalance:** only training data are balanced; test sets keep the real class ratio.
- **Models:** Logistic Regression, Decision Tree, Random Forest (100 trees), Gradient Boosting, with fixed settings and no hyperparameter search.
- **Metrics:** precision, recall, F1, false-positive rate (FPR), false-negative rate (FNR), ROC-AUC, PR-AUC (important for the imbalanced Impersonation data), macro-F1 for the four-class problem, and inference time per message.
- **Feature sets (ablation):**
  - *Full*: raw ID and bytes plus all engineered features
  - *Context*: engineered features only (no raw ID or bytes)
  - *Timing-only*: DLC and bus-wide timing only

---

## Results and figures

All figures are in the `figs/` folder and are generated by the scripts.

### A. Understanding the data

**Message rate in all four files** (`figs/12_rate_all_files.png`)

![Message rate over time in all four files](figs/12_rate_all_files.png)

Messages per second over time. Grey is all traffic and red is attack traffic. The attack-free capture is flat at about 2,300 messages/s. In the DoS file the load swings up to about 4,100 messages/s (the bus saturates) and about half of it is attack traffic. In the Fuzzy file the attack is a steady 500 to 1,200 messages/s. In the Impersonation file the attack is so small that the red line sits near zero.

**Attack share per file** (`figs/13_attack_share.png`)

![Share of attack messages in each file](figs/13_attack_share.png)

DoS is 51 % attack, Fuzzy 36 % and Impersonation only 0.57 %. This is why Impersonation needs special handling for imbalance and why PR-AUC is reported.

**What each attack looks like** (`figs/14_attack_signatures.png`)

![Signature of each attack](figs/14_attack_signatures.png)

Left: in normal traffic the busiest IDs each carry about 5 % of messages, but during DoS the single ID `0x000` carries about 51 %. Middle: fuzzed payloads cluster at the maximum entropy (about 3 bits, meaning nearly all 8 bytes differ), while normal payloads are spread out and often contain many zero bytes. Right: legitimate `0x164` messages arrive on a sharp 10 ms rhythm, while spoofed ones arrive at irregular, often much earlier times.

**DoS bus load** (`figs/02_rate_over_time.png`)

![DoS bus load over time](figs/02_rate_over_time.png)

Zoom on the DoS file. Total load (grey) varies between about 800 and 4,100 messages/s, and the attack messages (red) follow it at roughly half the total, interleaved with normal traffic for the whole capture.

**Normal traffic structure** (`figs/03_top_ids.png`) and **class balance** (`figs/01_class_balance.png`)

![Top normal CAN IDs](figs/03_top_ids.png)

The 15 most frequent normal IDs each send about 14,000 to 18,000 messages in the DoS capture, because ECUs transmit periodically. This regularity is what the per-ID features exploit. The class-balance plot (`figs/01_class_balance.png`) shows the DoS file is nearly balanced (about 336,000 attack vs 321,000 normal).

**Why bus-wide timing is not enough** (`figs/04_interarrival.png`)

![Inter-arrival time distribution](figs/04_interarrival.png)

Distribution of the time since the previous message on the bus (log scale). Normal and DoS messages overlap almost completely, because attack and normal messages are interleaved. This foreshadows the timing-only failure in the results below.

### B. Detection results (attack vs normal)

**DoS confusion matrices** (`figs/05_confusion_matrices.png`)

![DoS confusion matrices](figs/05_confusion_matrices.png)

Decision Tree results on the DoS test set for the three feature sets. With Full or Context features there are no errors at all. With timing-only features the model is close to guessing: it flags about half of the normal messages as DoS (51,332 of 97,220) and misses about 20,600 attack messages.

**Model comparison** (`figs/06_model_comparison.png`)

![F1 and FPR by model and feature set](figs/06_model_comparison.png)

F1 (left) and false-positive rate (right) for each model and feature set on DoS. All four models are perfect with Full and Context features and weak with timing-only features (F1 about 0.69, FPR 0.53 to 0.90). The feature set matters far more than the choice of model.

**ROC curve, timing-only** (`figs/07_roc_timing.png`)

![ROC curve for timing-only features](figs/07_roc_timing.png)

The ROC curve of a Random Forest using only timing features hugs the diagonal (AUC about 0.68), where the diagonal means random guessing.

**Summary across all three attacks** (`figs/11_summary_across_attacks.png`)

![Best F1 per attack and feature level](figs/11_summary_across_attacks.png)

The key figure of the project: best F1 over the models for each attack and feature level.

| Attack | Full | Context | Timing-only |
|---|---|---|---|
| DoS | 1.000 | 1.000 | 0.694 |
| Fuzzy | 1.000 | 0.941 | 0.620 |
| Impersonation | 1.000 | 1.000 | 0.596 |

For every attack, timing-only features perform poorly. Per-ID and payload information is what makes detection possible. Fuzzy is the only attack that stays below perfect without the raw ID.

### C. Four-class classification (Normal / DoS / Fuzzy / Impersonation)

**Confusion matrices** (`figs/08_multiclass_confusion.png`)

![Four-class confusion matrices](figs/08_multiclass_confusion.png)

Rows are the true class and columns the predicted class, with counts and row percentages. Left: with full features there are only 27 normal messages predicted as Fuzzy and 3 Fuzzy messages predicted as Normal. Right: without raw ID, bytes or bus timing, DoS and Impersonation are still classified without error, and the remaining mistakes are about 8,400 normal messages (1.7 %) predicted as Fuzzy. These come from one legitimate ID (`0x220`) whose normal payloads look random.

**Macro-F1 by model and feature set** (`figs/09_multiclass_macro_f1.png`)

![Macro-F1 by model and feature set](figs/09_multiclass_macro_f1.png)

Macro-F1 is the average F1 over the four classes. Note that the y-axis starts at 0.90, so differences look larger than they are. Full features give about 0.9999 (Decision Tree, Gradient Boosting), and without raw ID and bytes all models reach about 0.98.

**Per-class F1** (`figs/10_per_class_f1.png`)

![Per-class F1](figs/10_per_class_f1.png)

For the best model without raw ID and bytes: DoS and Impersonation reach F1 1.000, Normal 0.992 and Fuzzy 0.941. Fuzzy is the hardest class. (The y-axis starts at 0.90.)

| Feature set | Best model | Macro-F1 |
|---|---|---|
| A: Full (with raw ID and bytes) | Decision Tree | 0.9999 |
| B: Context (no raw ID/bytes) | Random Forest / Gradient Boosting | 0.9832 |
| C: No bus-wide timing | Random Forest / Gradient Boosting | 0.9832 |

**Session check.** Each attack comes from a different capture session. A classifier can tell which file a *normal* message came from with 83 to 89 % accuracy (chance is 33 %), so attack type and session are partly confounded (see Limitations).

### D. Anomaly detection and unseen attacks

Models trained on **normal traffic only** (autoencoder, Isolation Forest), and a Random Forest trained with one attack hidden and tested on it:

| Attack | Autoencoder (normal-only): F1 / FPR | Isolation Forest ROC-AUC | RF, attack **unseen**: recall | RF, attack **seen**: F1 |
|---|---|---|---|---|
| DoS | **0.997** / 0.006 | 0.966 | 0.000 | 1.000 |
| Fuzzy | 0.044 / 0.020 | 0.863 | 0.000 | 0.941 |
| Impersonation | 0.148 / 0.004 | 0.999 | 0.966 | 1.000 |

Supervised models do not generalise to unseen attack types. A normal-only autoencoder detects DoS (which changes the traffic statistics drastically) but not fuzzing. The Isolation Forest ranks attacks well but is sensitive to the alert threshold.

### E. Feature selection and cost

Features were ranked by Random Forest importance (training data only), and the four-class model was retrained on the top k:

| Top k features | Macro-F1 | F1 Fuzzy | F1 Impersonation |
|---|---|---|---|
| 3 | 0.796 | 0.912 | 0.284 |
| **5** | **0.983** | 0.941 | 1.000 |
| 8 | 0.983 | 0.941 | 1.000 |
| 13 (all) | 0.983 | 0.941 | 1.000 |

Five features (`id_count_win`, `bytes_changed_lag2`, `n_zero_bytes`, `byte_mean`, `byte_std`) match the accuracy of all 13. The k = 3 result is unstable (it depends on which near-tied feature ranks third), so only k of 5 or more should be trusted. Inference takes only about 0.02 to 1.3 microseconds per message for all models on a laptop CPU.

---

## Key findings

- **Bus-wide timing alone detects none of the attacks well** (F1 about 0.6 to 0.7). Per-ID and payload features are what matter.
- **DoS and Impersonation are detected essentially perfectly** with per-ID and payload features, even without the raw ID.
- **Fuzzy is the hardest attack:** some legitimate IDs (for example `0x220`) carry random-looking payloads, causing about 7.5 % false positives when the raw ID is not available.
- **Model choice matters little; the feature set matters a lot.** A Decision Tree matches larger ensembles at the lowest inference cost.
- **Supervised models fail on unseen attack types;** a normal-only autoencoder detects DoS but not fuzzing.
- **Five features are enough** to match the full feature set.

---

## How the project meets its objectives

| Objective | Where it is done |
|---|---|
| 1. Analyse and preprocess real CAN traffic | Parsing, labelling, analysis plots and features (steps 1 to 3, 6, 7, 9, 11, 16) |
| 2. Detect and classify intrusion types | Per-attack detection (steps 4, 8, 10) and four-class classification (step 12) |
| 3. Compare models with appropriate measures | Precision, recall, F1, FPR/FNR, ROC-AUC, PR-AUC, macro-F1 and inference time (all model steps) |
| Anomaly detection (keyword) and unseen attacks | Step 14 |
| Feature selection and computational cost | Step 15 |

---

## Project structure and how to run

```
.
├── step1_parse.py              # parse + label DoS
├── step2_eda.py                # DoS analysis plots
├── step3_features.py           # DoS features
├── step4_models.py             # DoS models
├── step5_plots.py              # DoS result plots
├── step6_fuzzy_label.py        # parse + label Fuzzy (uses attack-free data)
├── step7_fuzzy_features.py     # Fuzzy features
├── step8_fuzzy_models.py       # Fuzzy models
├── step9_imp_features.py       # parse + label + features for Impersonation
├── step10_imp_models.py        # Impersonation models
├── step11_build_all.py         # combine all attacks with one feature set
├── step12_multiclass.py        # four-class models + session check
├── step13_plots.py             # four-class and summary plots
├── step14_anomaly_unseen.py    # anomaly detection + unseen-attack test
├── step15_feature_selection.py # top-k feature selection
├── step16_eda_all.py           # analysis plots for all four files
├── requirements.txt
└── figs/                       # generated figures
```

**Setup**

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

**Run the pipeline in order** (from the project root, with the four dataset `.txt` files in place):

```bash
python3 step1_parse.py
python3 step2_eda.py
python3 step3_features.py
python3 step4_models.py
python3 step5_plots.py
python3 step6_fuzzy_label.py
python3 step7_fuzzy_features.py
python3 step8_fuzzy_models.py
python3 step9_imp_features.py
python3 step10_imp_models.py
python3 step11_build_all.py
python3 step12_multiclass.py
python3 step13_plots.py
python3 step14_anomaly_unseen.py
python3 step15_feature_selection.py
python3 step16_eda_all.py
```

Each script prints its results and saves CSV files and figures. Some steps are slow (parsing the 2.4M-line attack-free file, training the 12 to 16 model grids), so allow a few minutes each.

---

## Limitations

- **Heuristic labels** for Fuzzy and Impersonation. Scores measure agreement with our rules, not official ground truth.
- **Inflated scores with raw ID and bytes:** the labels are functions of them, so models can re-learn the labelling rule. The Context and Timing-only results are the more meaningful ones.
- **Session confounding:** each attack comes from a single capture session, and normal traffic from the three sessions can be told apart (83 to 89 % accuracy). Results do not show generalisation to other vehicles or sessions.
- **Simple attack patterns:** the Impersonation attack uses a fixed payload. A more realistic spoofing attack would be harder to detect.
- **Same-session evaluation:** test data are the later part of the same recordings used for training.
- **No hyperparameter tuning**, and a single time-based split.

---

## Reference

M. Sami, "Intrusion Detection in CAN bus," IEEE DataPort, Dec. 30, 2019, doi: 10.21227/24m9-a446.
