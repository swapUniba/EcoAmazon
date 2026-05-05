# 🌱 Eco-Amazon

> **Enriching E-commerce Datasets with Product Carbon Footprint for Sustainable Recommendations**

Eco-Amazon enriches three widely used Amazon review datasets — **Electronics**, **Clothing**, and **Home & Kitchen** — with item-level **Product Carbon Footprint (PCF)** estimates (kg CO₂e), generated via a zero-shot LLM prompting pipeline. The resource also ships a curated ground-truth subset of 159 products with official PCF values sourced from Environmental Product Declarations (EPDs), and a full sustainability-aware recommendation use case.

---

## Table of Contents

- [Overview](#overview)
- [Repository Structure](#repository-structure)
- [Datasets](#datasets)
- [Installation](#installation)
- [Usage](#usage)
  - [1. Download the base Amazon data](#1-download-the-base-amazon-data)
  - [2. Run the LLM estimation pipeline](#2-run-the-llm-estimation-pipeline)
  - [3. Train and evaluate the recommender system](#3-train-and-evaluate-the-recommender-system)
- [LLM Pipeline — Technical Details](#llm-pipeline--technical-details)
- [Recommender System Use Case](#recommender-system-use-case)
- [Ground Truth](#ground-truth)
- [Results](#results)
- [Citation](#citation)

---

## Overview

Most e-commerce benchmarks contain no environmental impact data, which severely limits research on sustainability-aware retrieval and recommendation. Eco-Amazon addresses this gap with three contributions:

1. **Enriched datasets** — the Amazon Electronics, Clothing, and Home & Kitchen metadata files, augmented with a `co2e_kg` field per item estimated by LLMs via zero-shot prompting.
2. **Estimation code** — reproducible scripts for Google Gemini 2.5 Flash (Vertex AI) and OpenAI o3-mini that can be applied to any product catalogue.
3. **Recommendation use case** — a full pipeline (training → PCF-aware re-ranking → evaluation) built on RecBole, showing the accuracy/sustainability trade-off under different α weights.

---

## Repository Structure

```
EcoAmazon/
├── ecoamazon/
│   ├── src/
│   │   ├── llm_estimation_gemini.py   # PCF estimation via Google Vertex AI (Gemini 2.5 Flash)
│   │   └── llm_estimation_openai.py   # PCF estimation via OpenAI (o3-mini) with cost tracking
│   └── data/                          # Enriched datasets (PCF-augmented metadata)
├── use_case/
│   ├── data/
│   │   └── process_data.py            # Converts Amazon review files to RecBole format
│   └── src/
│       ├── train_recsys.py            # HPO + training for BPR and LightGCN (Ray Tune)
│       ├── rerank_rec_list.py         # SaS re-ranking at multiple α values
│       ├── eval.py                    # Accuracy + sustainability + beyond-accuracy evaluation
│       └── utils.py                   # Checkpoint loading, asin↔item_id mapping
├── ground_truth_values/               # 600 products with official EPD-sourced PCF values
│   ├── gemini_2_5_flash/              # Esitimated predicted by Gemini-2.5-flash
│   ├── openai_o3_mini/                # Esitimated predicted by GPT-03-mini
└── README.md
```

---

## Datasets

### Enriched datasets (LLM-estimated PCF)

The ready-to-use enriched datasets are available on Zenodo: **[doi:10.5281/zenodo.18549130](https://doi.org/10.5281/zenodo.18549130)**

Each record extends the original Amazon metadata with:

| Field | Type | Description |
|---|---|---|
| `parent_asin` | string | Amazon product identifier |
| `co2e_kg` | float | Estimated PCF in kg CO₂ equivalent |
| `source` | string | `"EPD"` if an official declaration was found, otherwise `"LLM_estimated"` |
| `explanation` | string | Model's reasoning for the estimate |

> **Note:** The base Amazon metadata and review files are **not included** in this repository due to GitHub file size limits. See [Download the base Amazon data](#1-download-the-base-amazon-data) below.

### Ground truth (EPD-sourced PCF)

The `ground_truth_values/` folder contains **600 products** with PCF values manually collected from official Environmental Product Declarations. These are used to benchmark estimation quality. See [`ground_truth_values/README.md`](ground_truth/README.md) for the full schema and domain breakdown.

---

## Installation

```bash
git clone https://github.com/swapUniba/EcoAmazon.git
cd EcoAmazon
pip install -r requirements.txt
```

**Shared dependencies**
```
pandas
tqdm
python-dotenv
```

**Gemini pipeline (additional)**
```
google-cloud-aiplatform
google-auth
```

**OpenAI pipeline (additional)**
```
openai
```

**Recommender system use case (additional)**
```
recbole
torch
numpy
ray[tune]
```

> For RecBole setup details, see the [official RecBole documentation](https://recbole.io/docs/).

---

## Usage

### 1. Download the base Amazon data

The input files for both the estimation pipeline and the use case come from the **Amazon Reviews 2023** dataset ([amazon-reviews-2023.github.io](https://amazon-reviews-2023.github.io)).

**Metadata files** (input for PCF estimation):

| Domain | URL |
|---|---|
| Electronics | [meta_Electronics.jsonl.gz](https://mcauleylab.ucsd.edu/public_datasets/data/amazon_2023/raw/meta_categories/meta_Electronics.jsonl.gz) |
| Clothing | [meta_Clothing_Shoes_and_Jewelry.jsonl.gz](https://mcauleylab.ucsd.edu/public_datasets/data/amazon_2023/raw/meta_categories/meta_Clothing_Shoes_and_Jewelry.jsonl.gz) |
| Home & Kitchen | [meta_Home_and_Kitchen.jsonl.gz](https://mcauleylab.ucsd.edu/public_datasets/data/amazon_2023/raw/meta_categories/meta_Home_and_Kitchen.jsonl.gz) |

**Review files** (input for the use case):

| Domain | URL |
|---|---|
| Electronics | [Electronics.jsonl.gz](https://mcauleylab.ucsd.edu/public_datasets/data/amazon_2023/raw/review_categories/Electronics.jsonl.gz) |
| Clothing | [Clothing_Shoes_and_Jewelry.jsonl.gz](https://mcauleylab.ucsd.edu/public_datasets/data/amazon_2023/raw/review_categories/Clothing_Shoes_and_Jewelry.jsonl.gz) |
| Home & Kitchen | [Home_and_Kitchen.jsonl.gz](https://mcauleylab.ucsd.edu/public_datasets/data/amazon_2023/raw/review_categories/Home_and_Kitchen.jsonl.gz) |

### 2. Run the LLM estimation pipeline

**With Gemini (Vertex AI):**
```bash
# Set your service account key path inside llm_estimation_gemini.py
python ecoamazon/src/llm_estimation_gemini.py \
  --input data/meta_Electronics.jsonl \
  --output data/eco_Electronics.jsonl
```

**With OpenAI:**
```bash
# Create a .env file with: OPENAI_API_KEY=your_key
python ecoamazon/src/llm_estimation_openai.py \
  --input data/meta_Electronics.jsonl \
  --output data/eco_Electronics.jsonl
```

Both scripts support **resuming** interrupted runs — they skip already-processed `parent_asin` values automatically.

### 3. Train and evaluate the recommender system

```bash
# Step 1 — convert review files to RecBole format
python use_case/data/process_data.py

# Step 2 — copy processed files to dataset folder
cp use_case/data/processed/* use_case/src/dataset/

# Step 3 — hyperparameter search + train BPR / LightGCN
python use_case/src/train_recsys.py

# Step 4 — generate re-ranked lists at multiple α values
python use_case/src/rerank_rec_list.py

# Step 5 — evaluate accuracy + sustainability metrics
python use_case/src/eval.py
```

Results are written to `results.csv` for direct comparison across α values.

---

## LLM Pipeline — Technical Details

The estimation pipeline follows a two-step hierarchy for each product:

1. **Official data first** — the model searches for an Environmental Product Declaration (EPD) or manufacturer report. If found, the official PCF value is returned.
2. **LLM inference fallback** — if no official data exists, the model estimates CO₂e from product metadata (materials, weight, manufacturing process, transport) following ISO 14040/14044 and GHG Protocol standards.

Key engineering features:

- **Parallel processing** via `ThreadPoolExecutor` (default: 10 workers).
- **Resume logic** — already-processed items are skipped on restart.
- **Thread-safe writing** via `threading.Lock()` + `os.fsync()`.
- **Rate-limit handling** — exponential backoff on `429` errors (Gemini); retry logic for both backends.
- **Cost tracking** — the OpenAI script logs input/output tokens and USD cost per item.
- **Graceful shutdown** — `Ctrl+C` is caught and progress is preserved (Gemini script).

---

## Recommender System Use Case

The use case demonstrates how PCF scores can be integrated into a standard recommendation pipeline via post-hoc re-ranking.

**SaS (Sustainability-aware Score) formula:**

$$\text{SaS} = \alpha \cdot \text{Pred}_{norm} + (1 - \alpha) \cdot \text{PCF}_{norm}$$

- `α = 1.0` → standard accuracy-only ranking
- `α < 1.0` → increasing sustainability weight

Both `Pred` and `PCF` values are min-max normalised before combining. The pipeline evaluates α ∈ {1.0, 0.75, 0.5, 0.25} and reports:

- **Accuracy**: Recall@k, NDCG@k, MRR, Precision, F1
- **Sustainability**: Average PCF of top-k recommended items
- **Beyond-accuracy**: Gini Index, Average Popularity, Item Coverage

---

## Ground Truth

The `ground_truth_values/` folder contains a curated benchmark of **600 products** with PCF values sourced from official EPDs:

| Domain | Items |
|---|---:|
| Electronics | 200 |
| Clothing | 200 |
| Home & Kitchen | 200 |
| **Total** | **200** |

These are used to compute MAE and Spearman/NDCG scores against LLM estimates. See [`ground_truth_values/README.md`](ground_truth/README.md) for the full field schema.

