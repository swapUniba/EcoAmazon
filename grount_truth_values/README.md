# Ground Truth — EcoAmazon

This folder contains the **curated ground-truth dataset** used to evaluate the LLM-based Product Carbon Footprint (PCF) estimation pipeline described in the paper:

> *Eco-Amazon: Enriching E-commerce Datasets with Product Carbon Footprint for Sustainable Recommendations* 

---

## Purpose

Most products in large-scale e-commerce catalogs lack official PCF data, making it difficult to assess the quality of automated PCF estimation methods. This folder addresses that gap by providing a set of items for which **official, authoritative PCF values** were obtained from **Environmental Product Declarations (EPDs)** issued by manufacturers. These values serve as the reference signal against which LLM-generated estimates are benchmarked.

---

## Contents

The ground truth covers **600 products** across three Amazon product domains:

| Domain            | Items |
|-------------------|------:|
| Electronics       |   200 |
| Clothing          |   200 |
| Home and Kitchen  |   200 |
| **Total**         | **600** |

Each domain is stored in a separate file (one file per category). Files are structured as tabular data (CSV or JSON) where each row/entry represents one product.

---

## Data Fields

Each record is expected to contain at minimum the following fields:

| Field | Type | Description |
|---|---|---|
| `title` / `name` | string | Product title or name as listed on Amazon |
| `co2e_kg` | float | Official Product Carbon Footprint in **kg CO2e** (CO2 equivalent), sourced from the EPD or manufacturer report |
| `estimates` | list | List of estimates provided by the LLM |

> **Note:** The exact column names may vary slightly across files. Always refer to the header row of each file as the authoritative schema.

---

## Data Source & Collection

PCF values in this folder were **not** estimated by any model. They were manually collected from publicly available **Environmental Product Declarations (EPDs)** — standardised documents in which manufacturers officially report the environmental impact of their products, typically expressed in kg CO₂e over the full product lifecycle (cradle-to-gate or cradle-to-grave, depending on the EPD).

EPDs follow ISO 14025 and related Life Cycle Assessment (LCA) standards, making them the most credible publicly available source of PCF data for consumer goods.

---

## Role in the Evaluation Pipeline

This ground truth is used for two evaluation tasks:

1. **Absolute accuracy (MAE):** The Mean Absolute Error between the LLM-estimated PCF and the official EPD value is computed in kg CO₂e.
2. **Ranking quality (Spearman / NDCG):** Items within each category are ranked by their official PCF values (descending CO₂e emissions), and this ranking is compared to the ranking produced by the LLM estimates. Spearman correlation is normalised to the \[0, 1\] interval; NDCG is computed in the standard way.

This dual evaluation captures both absolute numerical precision and ordinal consistency — the latter being particularly important for sustainability-aware recommendation, where guiding users toward *relatively* greener alternatives matters even when absolute values are imprecise.


The full dataset is also available on Zenodo:
[https://doi.org/10.5281/zenodo.18549130](https://doi.org/10.5281/zenodo.18549130)
