# ML_Project
# CAN Network Intrusion Detection Using Machine Learning

Machine-learning detection and classification of attacks on the in-vehicle **Controller Area Network (CAN) bus**, using the public CAN Intrusion Dataset from IEEE DataPort (Mercedes ML350).

**Team:** Prateek Vamana Pai (240905006) · Dhrithi D Shetty (240905004) · Sameeksha Bajoria (240905190)

## Overview

The CAN bus was designed without authentication or encryption, so an attacker on the bus can inject malicious messages. This project:

1. Parses and labels raw CAN traffic (normal, DoS, Fuzzy and Impersonation attacks).
2. Engineers causal message-level and per-ID features.
3. Trains and compares four classifiers for **detection** (attack vs normal) and **classification** (which attack).
4. Tests **anomaly detection** (normal-only training) and **unseen-attack** generalisation.
5. Studies feature selection and inference cost.

## Dataset

M. Sami, "Intrusion Detection in CAN bus," IEEE DataPort, 2019. doi: [10.21227/24m9-a446](https://doi.org/10.21227/24m9-a446)
Download: https://ieee-dataport.org/documents/can-intrusion-dataset

The data files are **not included** in this repository. Download them and place them in the project root:

| File | Messages | Duration | Attack share (our labels) |
|---|---|---|---|
| `Attack_free_dataset.txt` | 2,369,398 | 1,037.6 s | 0 % |
| `DoS_attack_dataset.txt` | 656,579 | 306.3 s | 51.12 % |
| `Fuzzy_attack_dataset.txt` | 591,990 | 347.3 s | 36.23 % |
| `Impersonation_attack_dataset.txt` | 995,472 | 482.6 s | 0.57 % |

Line format: `Timestamp: 0.000271  ID: 0080  000  DLC: 8  00 17 dc 09 16 11 16 bb`

### Labelling (the files have no label column)

| Attack | Rule used to label attack messages |
|---|---|
| DoS | ID `0x000` (flooding, all-zero payload) |
| Fuzzy | ID with ≤ 20 distinct payloads in normal data, **and** the (ID, payload) pair never occurs in attack-free data, **and** it occurs ≤ 2 times in the Fuzzy file |
| Impersonation | ID `0x164` with the constant payload `01 02 03 04 05 06 07 08` |

> Fuzzy and Impersonation labels are heuristic, so results measure agreement with these rules, not official ground truth.

## Features

All features are causal (current and past messages only; window = last 100 messages).

- **Bus-wide timing:** `dt_prev`, `win_mean_dt`, `win_std_dt`
- **Per-ID behaviour:** `dt_same_id`, `bytes_changed`, `bytes_changed_lag2`, `id_count_win`, `dlc`
- **Payload statistics:** entropy, number of distinct bytes, byte mean and std, number of zero bytes
- **Raw message:** `can_id`, bytes `d0`–`d7` (used in the "Full" feature set only)

## Experimental setup

- Time-based split inside each file: first 70 % train, last 30 % test.
- Training sets balanced; test sets keep the real class ratio.
- Models: Logistic Regression, Decision Tree, Random Forest, Gradient Boosting (fixed settings, no hyperparameter search).
- Metrics: precision, recall, F1, FPR/FNR, ROC-AUC, PR-AUC, macro-F1, inference time.

## Results

### Detection, best F1 over models

| Attack | Full features | Context (no raw ID/bytes) | Timing-only |
|---|---|---|---|
| DoS | 1.000 | 1.000 | 0.694 |
| Fuzzy | 1.000 | 0.941 | 0.620 |
| Impersonation | 1.000 | 1.000 | 0.596 |

### Four-class classification (Normal / DoS / Fuzzy / Impersonation)

| Feature set | Best model | Macro-F1 |
|---|---|---|
| Full (with raw ID and bytes) | Decision Tree | 0.9999 |
| Context (no raw ID/bytes) | Random Forest / Gradient Boosting | 0.9832 |
| No bus-wide timing | Random Forest / Gradient Boosting | 0.9832 |

### Anomaly detection and unseen attacks

| Attack | Autoencoder (normal-only) F1 | RF, attack unseen: recall | RF, attack seen: F1 |
|---|---|---|---|
| DoS | 0.997 | 0.000 | 1.000 |
| Fuzzy | 0.044 | 0.000 | 0.941 |
| Impersonation | 0.148 | 0.966 | 1.000 |

### Feature selection

Five features (`id_count_win`, `bytes_changed_lag2`, `n_zero_bytes`, `byte_mean`, `byte_std`) match the macro-F1 of all 13 features (0.983).

## Key findings

- Bus-wide timing alone detects none of the attacks well (F1 ≈ 0.6–0.7); per-ID and payload features are what matter.
- DoS and Impersonation are detected essentially perfectly with per-ID/payload features.
- Fuzzy is the hardest attack: some legitimate IDs (e.g. `0x220`) carry random-looking payloads, causing ~7.5 % false positives without the raw ID.
- Model choice matters little; the feature set matters a lot. A Decision Tree matches larger ensembles at the lowest inference cost.
- Supervised models fail on unseen attack types; a normal-only autoencoder detects DoS but not fuzzing.

## Project structure and how to run

```
.
├── step1_parse.py             # parse + label DoS
├── step2_eda.py               # DoS analysis plots
├── step3_features.py          # DoS features
├── step4_models.py            # DoS models
├── step5_plots.py             # DoS result plots
├── step6_fuzzy_label.py       # parse + label Fuzzy (uses attack-free data)
├── step7_fuzzy_features.py    # Fuzzy features
├── step8_fuzzy_models.py      # Fuzzy models
├── step9_imp_features.py      # parse + label + features for Impersonation
├── step10_imp_models.py       # Impersonation models
├── step11_build_all.py        # combine all attacks with one feature set
├── step12_multiclass.py       # four-class models + session-leakage check
├── step13_plots.py            # multi-class and summary plots
├── step14_anomaly_unseen.py   # anomaly detection + unseen-attack test
├── step15_feature_selection.py# top-k feature selection
├── step16_eda_all.py          # analysis plots for all four files
└── figs/                      # generated figures
```

**Setup**

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

**Run the pipeline in order** (from the project root, with the four `.txt` files in place):

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

Each script prints its results and saves CSVs and figures. Several steps are slow (parsing the 2.4M-line attack-free file, training the 12–16 model grids), so allow a few minutes each.

## Limitations

- Heuristic labels for Fuzzy and Impersonation.
- Scores with raw ID and bytes are inflated, because the labels are functions of them.
- Each attack comes from a single capture session, and normal traffic from the three sessions can be told apart (83–89 % accuracy), so attack type and session are confounded. Results do not show generalisation to other vehicles.
- The Impersonation attack uses a fixed payload; a more realistic spoofing attack would be harder to detect.
- Evaluation uses later parts of the same recordings used for training.

## Reference

M. Sami, "Intrusion Detection in CAN bus," IEEE DataPort, Dec. 30, 2019, doi: 10.21227/24m9-a446.
