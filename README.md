# TRIVEX: Cascaded Intrusion Detection for IoMT Networks

TRIVEX is a lightweight, multi-stage intrusion detection system (IDS) for Internet of Medical Things (IoMT) network traffic. It combines fast gradient-boosted trees with a small 1D-CNN so that most of the work is done by cheap models. It was built as a final-year project (BS Computer Science, Government Postgraduate College Haripur, Pakistan) and evaluated on the CICIoMT2024 dataset.

> **Scope.** Everything here is offline evaluation on Google Colab. **TRIVEX has not been run on real edge hardware** (no Raspberry Pi or other device). Model size and CPU timings are proxies for edge suitability, not measurements on an edge device.

---

## Contents

1. [What is in this repo](#what-is-in-this-repo)
2. [Dataset](#dataset)
3. [How the system works](#how-the-system-works)
4. [Results](#results)
5. [Honest limitations](#honest-limitations)
6. [How to run](#how-to-run)
7. [Notebook map](#notebook-map)
8. [Security notes](#security-notes)
9. [Changes made before publishing](#changes-made-before-publishing)
10. [Suggested .gitignore](#suggested-gitignore)
11. [Citation and license](#citation-and-license)

---

## What is in this repo

| File | Purpose |
|---|---|
| `TRIVEX_Complete_Backend.ipynb` | The full pipeline in one Colab notebook: EDA, preprocessing, training, threshold tuning, evaluation, baselines, ONNX export, synthetic out-of-distribution test, and a demo Flask API |
| `README.md` | This file |

The dataset and trained model files are **not** included (see [Dataset](#dataset) and [.gitignore](#suggested-gitignore)).

---

## Dataset

- **CICIoMT2024** network-flow CSVs, 19 classes (1 benign, 18 attack types).
- Train: **1,826,608** rows. Test: **1,614,182** rows. Each file has 46 columns.
- The raw CSVs have no label column. Labels are derived from the **file names** (the notebook's cell C1 does this).
- 45 features are reduced to **30 selected features** before modeling.
- The 19 classes are merged into **11 classes** for the CNN (for example, DoS and DDoS variants of SYN, TCP, UDP and ICMP floods are merged, and the three larger recon types are merged).
- Get the dataset from its official source (Canadian Institute for Cybersecurity, University of New Brunswick) and follow its license and citation requirements. Place the files as described in [How to run](#how-to-run).

Class balance is very uneven. Some classes (for example Recon-Ping_Sweep with 740 training rows) are tiny, and the test set is about 97.7% attack traffic.

---

## How the system works

```
flow features (30)
      |
      v
Stage 1A: LightGBM binary (benign vs attack), threshold 0.25
      |-- attack --------------------------------------> Stage 2: 1D-CNN (11 classes)
      |
      '-- benign --> Stage 1B: Recon checker (LightGBM), threshold 0.05
                          |-- recon suspected ----------> Stage 2: 1D-CNN
                          '-- safe ---------------------> final label: Benign
```

| Component | Details |
|---|---|
| Stage 1A | LightGBM, 500 trees, binary objective. Threshold 0.25 chosen on the validation set to keep attack recall high (99.94% on validation) at the cost of more false alarms |
| Stage 1B | LightGBM "recon checker", 300 trees, trained on benign vs the four recon classes. It re-examines flows Stage 1A called benign. Threshold 0.05 |
| Stage 2 | 1D-CNN: Conv1D 64, 128, 256, then FC 128, then FC 11. 158,731 parameters, trained with focal loss (gamma 2) and class weights |
| Imbalance | SMOTE on the training split only, raising every class to at least 50,000 samples. The validation and test sets are untouched |
| Per-class filters | Extra thresholds for ARP spoofing (0.35) and ping sweep (0.75), tuned on the validation set |
| CNN confidence override | Tested; the best value was 0.0 (disabled) |
| Split | 90% train / 10% validation, stratified on the 19 classes, seed 42. The test set was used for the official evaluation after all thresholds were fixed |
| Compression | CNN exported to ONNX, then INT8 dynamic quantization (0.63 MB to 0.17 MB). **Only the CNN is quantized; the LightGBM models are not** |

### A note on how the LightGBM threshold was chosen

The first threshold search assumed the CNN would classify every flagged attack correctly (an "oracle"). That made low thresholds look better than they were. Cell E1 documents why this was wrong, discards those results, and uses a scan of real validation predictions instead.

---

## Results

All numbers below are from the notebook's saved outputs on the **official test set** (1,614,182 flows), unless stated otherwise.

### Main cascade (cell E5)

| Metric | Value |
|---|---|
| Accuracy | 99.574% |
| Macro-F1 | 0.8954 |
| Weighted-F1 | 0.9963 |
| Attack detection rate (TPR) | 99.94% |
| False alarm rate (FPR) | 7.64% |
| MCC | 0.9945 |
| TP / TN / FP / FN | 1,575,674 / 34,735 / 2,872 / 901 |

### Per-class results (11 classes)

| Class | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| ARP_Spoofing | 0.23 | 0.60 | 0.33 | 1,744 |
| Benign | 0.97 | 0.92 | 0.95 | 37,607 |
| ICMP_Flood | 1.00 | 1.00 | 1.00 | 448,131 |
| MQTT-Malformed_Data | 0.72 | 0.91 | 0.81 | 1,747 |
| MQTT_Connect_Flood | 1.00 | 1.00 | 1.00 | 45,047 |
| MQTT_Publish_Flood | 0.97 | 1.00 | 0.98 | 16,921 |
| Recon-Ping_Sweep | 0.92 | 0.76 | 0.83 | 186 |
| Recon_Scan | 0.98 | 0.93 | 0.96 | 27,490 |
| SYN_Flood | 1.00 | 1.00 | 1.00 | 270,992 |
| TCP_Flood | 1.00 | 1.00 | 1.00 | 264,694 |
| UDP_Flood | 1.00 | 1.00 | 1.00 | 499,623 |

The large flood classes are near perfect. The weak spots are **ARP spoofing** (many false positives) and the small classes.

### Baselines (cell F2)

Same 30 features, same SMOTE training data, same test set.

| Model | Accuracy | Macro-F1 | TPR | FPR | Model size |
|---|---|---|---|---|---|
| Random Forest | 99.71% | 0.8962 | 99.94% | 4.00% | 156.3 MB |
| XGBoost (multiclass) | 99.75% | 0.9117 | 99.94% | 3.27% | 11.3 MB |
| LightGBM (single multiclass) | 99.78% | **0.9295** | 99.96% | **2.57%** | 22.5 MB |
| **TRIVEX cascade** | 99.57% | 0.8954 | 99.94% | 7.64% | **5.7 MB** |

**Read this honestly:** the cascade is the smallest model, but a single multiclass LightGBM scores higher on macro-F1 and has fewer false alarms. The cascade trades accuracy for size.

### Ablation (cell F1)

| Configuration | Macro-F1 | TPR | FPR | Share of flows reaching the CNN |
|---|---|---|---|---|
| CNN only | 0.8685 | 99.95% | 19.35% | 100% |
| LightGBM then CNN | **0.9000** | 99.94% | 6.00% | 97.8% |
| Full cascade (with recon checker) | 0.8954 | 99.94% | 7.64% | 97.9% |

Two things stand out. First, the recon checker slightly **lowers** macro-F1 and raises false alarms compared with LightGBM then CNN. Second, on this attack-heavy test set almost every flow reaches the CNN, so the compute saving is small here. The saving would be larger on realistic traffic that is mostly benign. That claim is **not tested** in this repo.

### Out-of-distribution test (cells H1B and H2)

A two-layer anomaly check (raw feature range plus prediction entropy) was tested on **1,500 synthetic samples that I generated** (mixed protocol, header manipulation, timing channel) versus 5,186 known attack samples: TPR 85.7%, FPR 0.2%, AUROC 0.8735.

This is **not** zero-day detection on real unknown attacks. The synthetic samples were made by me, and the thresholds were tuned on the same samples, so the numbers are optimistic. Treat it as a proof of concept only.

---

## Honest limitations

- **No real edge hardware.** Not run on a Raspberry Pi or any edge device. Model size (5.7 MB total for the cascade) and CPU timings are proxies only. An earlier version of cell G1 printed a "Raspberry Pi 4 estimate" built from an assumed slowdown factor. That was not a measurement and has been removed.
- **False alarm rate rose from validation to test.** FPR was about 3.6% on validation and 7.64% on test. The cascade does not generalize as cleanly to the test distribution as the validation numbers suggested.
- **Weak classes.** ARP spoofing (F1 0.33), and to a lesser extent the small MQTT-Malformed and Ping-Sweep classes.
- **CNN is modest on its own.** Its validation macro-F1 was about 0.81 at the best epoch.
- **LightGBM Stage 1A hit its 500-tree limit** without early stopping, so it may be slightly under-trained.
- **Hand-made sanity test failed.** In cell D1B, a hand-built "normal web browsing" packet was classified as an attack. This synthetic check is not a reliable test, but it is a reminder that the model is trained on one dataset's feature distribution.
- **Single dataset, single split.** No cross-dataset or cross-device evaluation. No repeated runs with different seeds, so no confidence intervals.
- **Quantization check needs a re-run.** The original accuracy-after-compression check (G1B) used the first 50,000 rows of the unshuffled test set, which are mostly a few classes. Its numbers (81.9% accuracy) are not valid and were removed. The notebook now uses a random sample; re-run cells G1 and G1B and record the new numbers here.
- **Throughput numbers are not comparable across tables.** They were measured on Colab in different runs and on different devices (GPU and CPU), so use them only as rough indications.

---

## How to run

**Requirements:** a Google account, Google Colab (a T4 GPU is enough; CNN training uses it, most other steps run on CPU), and Google Drive for storing data and artifacts.

1. Download CICIoMT2024 and place the CSVs in this Drive layout:
   ```
   MyDrive/IOMT_IDS_Project/
     dataset/train/*.csv
     dataset/test/*.csv
   ```
   The notebook creates `models/`, `results/`, `processed/`, `plots/` and `onnx/` automatically.
2. Upload `TRIVEX_Complete_Backend.ipynb` to Colab and set the runtime to GPU.
3. Run cells **A1** (installs and mounts Drive) then **A2** (paths and artifact check).
4. Either:
   - run everything in order (B, C, D, E, F, G, H, I) for a fresh start, or
   - if artifacts already exist on Drive, run the **Quick Restore** cell instead of re-training.
5. Cells that touch the final test set are marked. **E5 is meant to be run once**; thresholds are fixed from validation data before it runs.
6. After editing, re-run **G1** and **G1B** to regenerate the ONNX/quantization numbers and the single-thread CPU latency.

Some cells exist in two versions (for example "FAST RELOAD" variants of C3, C4, C5 and H1). Use the reload versions only when the saved arrays already exist on Drive.

---

## Notebook map

| Section | Cells | What it does |
|---|---|---|
| A | A1, A2, Quick Restore | Environment, paths, load saved artifacts |
| B | B1 to B3 | Data quality, class distribution, feature analysis |
| C | C1 to C9 | Labeling from file names, cleaning, capping outliers, 19 to 11 class merge, feature selection, split, scaling, SMOTE, saving arrays |
| D | D1, D1B, D2, D3, D3B | Train LightGBM (Stage 1A), recon checker (1B), CNN (Stage 2), first cascade test on validation data |
| E | E1 to E5 | Threshold selection on validation data, final one-shot test evaluation |
| F | F1 to F5 | Ablation, baselines, threshold sensitivity, SHAP, CNN override ablation |
| G | G1, G1B | ONNX export, INT8 quantization, size and CPU latency, compression check |
| H | H1, H1B, H2 | Synthetic out-of-distribution detection |
| I, J, K | I1 to I3, J1, F4C, K1 | Inference functions, synthetic data generator, master reload, SHAP for the demo, demo Flask API |

---

## Security notes

- **No secrets in the notebook.** An earlier version contained a tunnel service auth token in plain text. It has been removed. If you ever pasted a real token in a cell, revoke it in the provider's dashboard; deleting it from the file is not enough once it has been shared or committed.
- **The Flask demo API (cell K1) is for demos only.** It has no authentication, and CORS is open. While the Cloudflare tunnel runs, anyone with the temporary URL can call it. Do not run it while handling real network data, and stop it when finished.
- Do not commit the dataset, Drive paths with personal information, or any `.env` or credentials files.

---

## Changes made before publishing

- Removed the plain-text ngrok authtoken cell (the demo uses a Cloudflare tunnel instead).
- Removed saved outputs that contained a temporary public tunnel URL.
- Removed the invented Raspberry Pi 4 throughput estimate and the "fits on Pi 4" and "deploy on Pi 4" lines. Replaced them with a measured single-thread CPU latency check (Colab CPU, not a Pi).
- Fixed the compression check (G1B) to use a random fixed-seed sample instead of the first rows of the test set. Its outputs and G1's were cleared and must be regenerated by re-running those cells.
- Removed a saved `KeyboardInterrupt` error output from the setup cell.
- Softened an early comment claiming the LightGBM model "fits on Raspberry Pi easily" to say it has not been tested on edge hardware.
- Added a scope note at the top of the notebook.

The edited notebook was syntax-checked but **not re-executed** (it needs the dataset and a Colab GPU). Results in this README come from the original saved outputs, except where marked as needing a re-run.

---

## Suggested .gitignore

```
# data and large artifacts
dataset/
processed/
*.npz
*.pth
*.onnx
*.joblib

# secrets
.env
*.key

# notebooks checkpoints
.ipynb_checkpoints/
```

---

## Citation and license

- Cite the CICIoMT2024 dataset according to its official page.
- Add a `LICENSE` file of your choice (for example MIT) before publishing.
- Author: Eman Aslam. 
