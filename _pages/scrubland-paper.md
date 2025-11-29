---
layout: page
title: "Distinguishing Scrublands, Farms, and Plantations Across India"
description: "Motivation and methodology for our multi-resolution LULC mapping framework using high-resolution CV-derived ground truth."
date: 2025-11-28 10:00:00 +0530
categories: [research]
tags: [lulc, scrubland, agriculture, plantations, remote-sensing]
permalink: /scrubland-paper/
---

**Authors:**  
Raman Kumar, Aatif Dar, Aaditeshwar Seth  
*Department of Computer Science, IIT Delhi*

# Introduction
Scrublands are ecologically and socially important landscapes in India, supporting biodiversity, groundwater recharge, and pastoral livelihoods. Yet they are often misclassified as wastelands, causing them to be overlooked in restoration and land-management programs. A major challenge is that scrublands and rain-fed agricultural fields exhibit **very similar spectral and seasonal patterns**, making them difficult to distinguish in existing global LULC datasets.

Similarly, plantations pose an additional difficulty because most global and national datasets merge them into generic tree or agricultural classes. This affects assessments related to deforestation, afforestation drives, and cropping-intensity measurements.

These problems exist largely due to the **lack of high-quality, India-specific labeled datasets** for scrublands, farms, and plantations. Models trained few global datasets using medium-resolution imagery struggle to capture India’s fragmented agro as well as non-agro(scrublands, plantations, etc) landscapes.

To address this, our paper develops a generalized **computer-vision-based methodology** that uses **high-resolution (1 m) imagery** to automatically generate large-scale, high-quality training samples. We used this approach to generate samples for farms, scrublands, and plantations. These samples are then used to train improved classifiers at 10 m using Googles satelite embeddings v1, enabling more accurate differentiation and creating India’s first nationwide plantation layer.  
*[Paper draft](https://ramank1137.github.io/assets/pdf/Scrubland_vs_Farmland_vs_Plantation.pdf)*


# Methodology Overview

The workflow contains **five sequential components**, summarized below.

><div style="background:white; padding:10px; display:block; text-align:center;"> <img src="/assets/img/Pipeline.png" alt="Pipeline" style="max-width:100%; height:auto;"> </div>
> The figure shows the overview of five module pipeline. (1) Spatial Representativeness and Tile Selection: Select representative AEZ grids using Jensen–Shannon divergence. (2) High-Resolution Boundary Delineation: Extract farm, scrubland, and plantation boundaries using CV models on 1 m imagery. (3) Rule-Based Boundary Refinement: Apply entropy, rectangularity, and size thresholds to retain high-confidence segments. (4) Sample Generation and Classifier Training: Generate samples and train AEZ-specific Random Forest classifiers using 64-dim embeddings. (5) Integration into IndiaSAT: Combine AEZ predictions with IndiaSAT modules to produce final farm, scrubland, and plantation classes.

# 1. Spatial Representativeness and Tile Selection

Each Agro-Ecological Zone (AEZ) is divided into grids:

$$
G = \{ g_1, g_2, \dots, g_N \}
$$

For each grid $g_i$, a normalized feature distribution $P_i$ is derived from clusters of Google's 64-dimensional embeddings.

The task is to select a subset:

$$
S \subset G, \quad |S| = p
$$

such that its aggregated distribution:

$$
P_S = \frac{1}{|S|} \sum_{g_i \in S} P_i
$$

approximates the AEZ-level distribution:

$$
P_{\text{AEZ}} = \frac{1}{N} \sum_{g_i \in G} P_i
$$

Selection minimizes the **Jensen–Shannon (JS) divergence)**:

$$
D_{JS}(P \parallel Q)
= \frac{1}{2} D_{KL}(P \parallel M)
+ \frac{1}{2} D_{KL}(Q \parallel M),
\quad
M = \frac{1}{2}(P + Q)
$$

At each iteration:

$$
g^* =
\arg\min_{g_i \in G \setminus S}
D_{JS} \Bigl( P_{\text{AEZ}} \parallel P_{S \cup \{ g_i \}} \Bigr)
$$

Approximately **3% of grids per AEZ** are selected and subdivided into 16×16 sub-grids.

><div style="background:white; padding:10px; display:block; text-align:center;"> <img src="/assets/img/Representative-tile.png" alt="Pipeline" style="max-width:100%; height:auto;"> </div>

# 2. High-Resolution Boundary Delineation

High-resolution imagery (zoom 17) is downloaded for each selected block.

Two CV pipelines operate on these tiles:

## 2.1 Farm & Non-Agricultural Boundaries

A **FracTAL-ResUNet** model generates:

- field extent  
- boundary probability  
- distance-to-boundary  

These outputs are merged and processed by **hierarchical watershed segmentation** to obtain closed polygons.

For each segment:

### Entropy

$$
H = - \sum_{i=1}^{L} p_i \log_2(p_i)
$$

Segment-level entropy:

$$
\overline{H}_S
= \frac{1}{|S^+|}
\sum_{x \in S^+} H(x)
$$

### Rectangularity

$$
R = \frac{A_{\text{contour}}}{A_{\text{rect}}}
$$

### Size  
Computed as pixel-area of the segment.

## 2.2 Plantation Boundaries

Plantations are detected using a **fine-tuned YOLO model**, retaining only high-confidence detections.

# 3. Rule-Based Boundary Refinement

High-confidence segments are identified using the thresholds defined in the paper.

## Farm rules

- Entropy < 1.0  
- Rectangularity > 0.67  
- Size ∈ [500, 2000] m²  
- Must belong to a cluster of ≥ 3 farms

## Scrubland / non-agro rules

- Size ∈ [60,000, 5,000,000] m²  
- >50% non-agricultural pixels (via IndiaSAT v3)

## Plantation rules

- Size ∈ [1,000, 20,000] m²

## Overlap hierarchy

$$
\text{Plantation} > \text{Farm} > \text{Scrubland}
$$

# 4. Sample Generation and Classifier Training

- ~150 samples per class are extracted per 16×16 block  
- For each sample, **Google’s 64-dimensional embeddings** for the **past 3 years** are collected  

A **Random Forest classifier is trained for each AEZ**, allowing regional adaptation and avoiding over-generalization.

The resulting models produce farm, non-agro, and plantation predictions at **10 m spatial resolution**.

# 5. Integration into IndiaSAT

The AEZ-level outputs are merged with the IndiaSAT v3 framework:

1. Add built-up, water, and barren classes  
2. Insert farm, non-agro, and plantation predictions  
3. Split farms into crop-intensity classes (8, 9, 10, 11)  
4. Split non-agro into forest vs. non-forest; non-forest is scrubland (12)  
5. Assign plantations as class (13)

# Conclusion

This methodology enables accurate mapping of scrublands, farms, and plantations across India by combining high-resolution CV-derived boundaries with AEZ-specific modeling. It produces India’s first nationwide plantation layer and significantly improves distinction between scrublands and rain-fed agriculture.

