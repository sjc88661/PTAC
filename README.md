<div align="center">

# PTAC

**Protein Thermal Adaptation Classifier (PTAC)**

*A dual-backbone fusion model for classifying proteins into mesophiles, thermophiles, and psychrophiles.*

</div>

---

<details open>
<summary><b>Contents</b></summary>

- [Overview](#overview)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Performance](#performance)
- [Project Structure](#project-structure)
- [Usage](#usage)
  - [Download Backbone Models](#1-download-backbone-models-required)
  - [Train](#2-train)
  - [Test](#3-test)
  - [Predict](#4-predict)
- [Released Weights](#released-weights)
- [Notes](#notes)

</details>

---

## Overview
### Dataset

The dataset used in this study consists of protein sequences annotated with thermal adaptation classes.

Each sequence is assigned to one of three classes:

| Label | Class |
|---|---|
| 0 | Mesophile |
| 1 | Thermophile |
| 2 | Psychrophile |

The manuscript describes a 2,400-sequence three-class dataset divided into 1,920 training and 480 test records. In a file-level audit, a candidate dataset with those counts matched the class-labelled sequence collection published by Huang et al. The original collection did not specify a train/test split; whether the candidate files and split correspond to the manuscript's final run and released checkpoints remains to be confirmed.

Source: Huang A, Lu F, Liu F. Discrimination of psychrophilic enzymes using machine learning algorithms with amino acid composition descriptor. *Frontiers in Microbiology*. 2023;14:1130594. [DOI: 10.3389/fmicb.2023.1130594](https://doi.org/10.3389/fmicb.2023.1130594). The published source contains some identical sequences with conflicting class labels; these require review rather than an assumed correction.

The repository currently contains:
```
data/
├── train.csv
└── test.csv
```

> **Data availability notice:** The current `data/train.csv` contains 1,495 complete records and one truncated record, so it does not contain the complete 1,920-record candidate training set. `data/test.csv` contains 480 complete records and matched the candidate test set by ID, sequence, and label in the audit. The training file and its relationship to the final manuscript run require confirmation before reproducible training. The published files have not been silently replaced.

**PTAC** predicts a protein's thermal adaptation class from its amino acid sequence:

| Label | Class | Description |
|:-----:|:------|:------------|
| `0` | **Mesophile**　嗜温 | Optimal growth at moderate temperatures (~20–45°C) |
| `1` | **Thermophilic**　嗜热 | Optimal growth at high temperatures (~45–80°C) |
| `2` | **Psychrophilic**　嗜冷 | Optimal growth at low temperatures (~-15-20°C) |

### Architecture

PTAC combines **two frozen protein language models** with a **trainable BiGRU fusion head**:

| Component | Model | Hidden Dim |
|:----------|:------|:----------:|
| Backbone 1 | [`ProtT5-XL-UniRef50`](https://huggingface.co/Rostlab/prot_t5_xl_uniref50) | 1024 |
| Backbone 2 | [`ProPrime_650M_OGT_Prediction`](https://huggingface.co/AI4Protein/ProPrime_650M_OGT_Prediction) | 1280 |
| Fusion Head | BiGRU → Attention Pooling → MLP | 512 |

> **Why two models?** ProtT5 provides general-purpose protein representations, while ProPrime is specifically pre-trained for optimal growth temperature (OGT) prediction. Their token-level hidden states are fused via BiGRU to capture complementary signals.

---

## Installation

### Requirements

- **Python** ≥ 3.9
- **CUDA-capable GPU** recommended (CPU supported but slow)
- **Disk** ~8 GB for backbone models

```bash
git clone https://github.com/sjc88661/PTAC.git
cd PTAC

# Install dependencies
pip install -r requirements.txt

# Download backbone models (~7.5 GB)
# (If you're in a region with limited Hugging Face access, use the mirror:)
#   Windows:  set HF_ENDPOINT=https://hf-mirror.com && python scripts/download_backbones.py
#   Linux:    HF_ENDPOINT=https://hf-mirror.com python scripts/download_backbones.py
python scripts/download_backbones.py
```

---

## Quick Start

Predict thermal classes for your own sequences in 3 steps:

```bash
# 1. Prepare input (CSV with columns: id,seq)
#    Linux: head examples/example_test.csv
#    Windows: Get-Content examples/example_test.csv -Head 5

# 2. Run prediction
python predict/predict.py \
  --input_path examples/example_test.csv \
  --input_type csv \
  --ckpt_dir train/fusion_out \
  --out_dir my_predictions

# 3. View results
#    Linux: cat my_predictions/predictions_ensemble.csv
#    Windows: Get-Content my_predictions/predictions_ensemble.csv
```

> Supports **CSV** and **FASTA** input. See [Predict](#4-predict) for full options.

---

## Performance

### Manuscript-reported independent-test results

The values below reproduce the manuscript's Table 1 and Supplementary Table S1 at their reported precision. The manuscript's ACC, macro F1 and MCC match the rounded averages in a candidate summary of five models; those averages should not be interpreted as metrics from probability-averaged ensemble predictions. The link between the manuscript run, the candidate result files, and the released checkpoints is still being verified.

| Metric (Table 1) | Manuscript value |
|:---|---:|
| Accuracy (ACC) | 94.9% |
| Macro F1 | 94.8% |
| MCC | 92.5% |
| Specificity (SP) | 95.0% |
| Precision (PRE) | 94.8% |
| AUROC | 99.2% |
| AUPRC | 98.0% |

| Class (Supplementary Table S1) | F1 |
|:---|---:|
| Mesophilic (嗜温) | 92.2% |
| Thermophilic (嗜热) | 98.9% |
| Psychrophilic (嗜冷) | 93.3% |

### Repository summary and unresolved differences

The existing [`results/test_metrics_summary.csv`](results/test_metrics_summary.csv) records the following means. It is retained as an existing result file, not rewritten to match the manuscript. The manuscript's Table 1 reports SP as 95.0% and PRE as 94.8%, whereas this summary reports macro SP as 97.4962% and macro PRE as 94.9824%. The manuscript's Supplementary Table S1 reports PRE as 95.0%. These differences and the exact experiment/checkpoint provenance require verification before choosing a single value.

| Metric | Repository summary mean |
|:---|---:|
| ACC | 0.949167 |
| MCC | 0.924808 |
| Macro sensitivity | 0.948245 |
| Macro specificity | 0.974962 |
| Macro precision | 0.949824 |
| Macro F1 | 0.947853 |
| Macro AUROC | 0.991738 |
| Micro AUROC | 0.993130 |
| Macro AUPRC | 0.980289 |
| Micro AUPRC | 0.987346 |

The previously displayed per-class F1 values 95.88%, 94.31%, and 94.18% did not have a traceable source in the records inspected and have been replaced above by the manuscript-reported values. Visual artifacts are available in [`results/`](results/); their connection to the manuscript run remains to be confirmed.

---

## Project Structure

```text
PTAC/
├── data/                         # Training and independent test datasets
├── examples/                     # Example input and prediction output
├── predict/
│   └── predict.py                # Predict thermal adaptation classes
├── results/                      # Evaluation results and case studies
├── scripts/
│   └── download_backbones.py     # Download the two backbone models
├── test/
│   └── test.py                   # Evaluate the trained models
├── train/
│   ├── train.py                  # Train the fusion head
│   └── fusion_out/               # Released fusion head checkpoints
├── requirements.txt
└── README.md
```

---

## Usage

### 1. Download Backbone Models (Required)

```bash
python scripts/download_backbones.py
```

This creates two local directories:

| Directory | Source | Size |
|:---|:---|---:|
| `prot_t5_xl_uniref50/` | `Rostlab/prot_t5_xl_uniref50` | ~5 GB |
| `ProPrime_650M_OGT_Prediction/` | `AI4Protein/ProPrime_650M_OGT_Prediction` | ~2.5 GB |

> ⚠️ These directories are **not** included in the repository due to size limits.

### 2. Train

Train the fusion head with 5-fold cross-validation:

```bash
python train/train.py \
  --train_csv data/train.csv \
  --prott5_model_path prot_t5_xl_uniref50 \
  --proprime_model_path ProPrime_650M_OGT_Prediction \
  --mode bigru-fusion \
  --out_dir train/fusion_out
```

<details>
<summary><b>Key training options</b></summary>

| Flag | Default | Description |
|:---|:---|:---|
| `--epochs` | 20 | Max training epochs |
| `--patience` | 7 | Early stopping patience |
| `--lr` | 1e-4 | Learning rate |
| `--batch_size` | 8 | Batch size per GPU |
| `--k_folds` | 5 | Number of CV folds |
| `--fp16` | — | Enable AMP mixed precision |
| `--use_cache` | — | Cache backbone embeddings to disk |
| `--tensorboard` | — | Log to TensorBoard |

</details>

### 3. Test

Evaluate all 5 folds on the test set:

```bash
python test/test.py \
  --test_csv data/test.csv \
  --train_csv data/train.csv \
  --prott5_model_path prot_t5_xl_uniref50 \
  --proprime_model_path ProPrime_650M_OGT_Prediction \
  --mode bigru-fusion \
  --ckpt_dir train/fusion_out \
  --out_dir results/test_out
```

Output includes per-fold and ensemble metrics, ROC/PR curves, confusion matrices, and t-SNE/UMAP visualizations.

### 4. Predict

```bash
# From CSV
python predict/predict.py \
  --input_path your_sequences.csv \
  --input_type csv \
  --ckpt_dir train/fusion_out \
  --prott5_model_path prot_t5_xl_uniref50 \
  --proprime_model_path ProPrime_650M_OGT_Prediction \
  --mode bigru-fusion \
  --out_dir pred_output

# From FASTA
python predict/predict.py \
  --input_path your_sequences.fasta \
  --input_type fasta \
  --ckpt_dir train/fusion_out \
  --out_dir pred_output
```

**Output files:**

| File | Description |
|:---|:---|
| `predictions_per_fold.csv` | Per-fold probabilities and predictions |
| `predictions_ensemble.csv` | **Mean-ensemble** probabilities and final prediction |

---

## Released Weights

The repository includes 5 pre-trained fusion head checkpoints:

```
train/fusion_out/
├── best_head_fold_1.pt    (13.5 MB)
├── best_head_fold_2.pt    (13.5 MB)
├── best_head_fold_3.pt    (13.5 MB)
├── best_head_fold_4.pt    (13.5 MB)
└── best_head_fold_5.pt    (13.5 MB)
```

Each checkpoint contains both the model `state_dict` and training `args` for reproducibility.

---

## Notes

- The two backbone models (`ProtT5` / `ProPrime`) are **not included** — download them via `scripts/download_backbones.py`.
- `results/` contains curated summary artifacts; full raw evaluation outputs are generated by `test/test.py`.
- Input can be **CSV** (`id,seq`) or **FASTA** format.
- For large-scale prediction, enable `--use_cache` and `--fp16` to reduce memory usage and speed up repeated runs.

---

