---
layout: page
title: "Scrubland–Farm–Plantation Paper"
permalink: /scrubland-paper/
---

---
layout: page
title: "Distinguishing Scrublands, Farms, and Plantations Across India"
description: "Motivation and methodology for our multi-resolution LULC mapping framework using high-resolution CV-derived ground truth."
date: 2025-11-28 10:00:00 +0530
categories: [research]
tags: [lulc, scrubland, agriculture, plantations, remote-sensing]
permalink: /blog/lulc-v4-methodology/
---

**Authors:**  
Raman Kumar, Aatif Dar, Aaditeshwar Seth  
*Department of Computer Science, IIT Delhi*

---

# Introduction

Scrublands are ecologically and socially important landscapes in India, supporting biodiversity, groundwater recharge, and pastoral livelihoods. Yet they are often misclassified as wastelands, causing them to be overlooked in restoration and land-management programs. A major challenge is that scrublands and rain-fed agricultural fields exhibit **very similar spectral and seasonal patterns**, making them difficult to distinguish in existing global LULC datasets.

Similarly, plantations pose an additional difficulty because most global and national datasets merge them into generic tree or agricultural classes. This affects assessments related to deforestation, afforestation drives, and cropping-intensity measurements.

These problems exist largely due to the **lack of high-quality, India-specific labeled datasets** for scrublands, farms, and plantations. Models trained few global datasets using medium-resolution imagery struggle to capture India’s fragmented agro as well as non-agro(scrublands, plantations, etc) landscapes.

To address this, our paper develops a generalized **computer-vision-based methodology** that uses **high-resolution (1 m) imagery** to automatically generate large-scale, high-quality training samples. We used this approach to generate samples for farms, scrublands, and plantations. These samples are then used to train improved classifiers at 10 m using Googles satelite embeddings v1, enabling more accurate differentiation and creating India’s first nationwide plantation layer.  
*[Paper draft](https://ramank1137.github.io/assets/pdf/Scrubland_vs_Farmland_vs_Plantation.pdf)*

---

# Methodology Overview

The workflow consists of **five sequential components**, shown in Figure 1 of the paper.

> **Figure Placeholder:**  
> _Insert Figure 1: Complete pipeline overview._

---

# 1. Spatial Representativeness and Tile Selection

Each Agro-Ecological Zone (AEZ) is divided into a set of grids:

\[
G = \{ g_1, g_2, \dots, g_N \}
\]

where each grid corresponds to a 9 km × 9 km region containing 32×32 high-resolution tiles.

For every grid \( g_i \), we compute a normalized feature distribution \( P_i \) using clusters derived from Google’s 64-dimensional embeddings. The objective is to select a subset:

\[
S \subset G, \quad |S| = p
\]

such that the aggregated distribution:

\[
P_S = \frac{1}{|S|} \sum_{g_i \in S} P_i
\]

approximates the AEZ-level distribution:

\[
P_{\text{AEZ}} = \frac{1}{N} \sum_{g_i \in G} P_i
\]

Selection is driven by minimizing the **Jensen–Shannon divergence**:

\[
D_{JS}(P \parallel Q) = \frac{1}{2} D_{KL}(P \parallel M) + \frac{1}{2} D_{KL}(Q \parallel M), \quad
M = \frac{1}{2}(P+Q)
\]

At each step, the grid added is:

\[
g^* = \arg\min_{g_i \in G \setminus S}
D_{JS}\left( P_{\text{AEZ}} \parallel P_{S \cup \{g_i\}} \right)
\]

Approximately **3% of grids per AEZ** are selected. These are subdivided into 16×16 sub-grids for high-resolution processing.

---

# 2. High-Resolution Boundary Delineation

High-resolution imagery (zoom level 17) is downloaded for each representative block. Two computer-vision pipelines operate on these tiles:

## 2.1 Farm & Non-Agricultural Boundaries

We apply a **FracTAL-ResUNet** model (following Wang et al.) that predicts:

- field extent  
- boundary probability  
- distance-to-boundary  

Because the model outputs probability maps, **hierarchical watershed segmentation** is used to obtain closed polygons over the stitched 16×16 outputs.

For each boundary, we compute:

### **Entropy**
Local Shannon entropy over grayscale neighborhoods:

\[
H = -\sum_{i=1}^L p_i \log_2(p_i)
\]

Segment-level entropy:

\[
\overline{H}_S = \frac{1}{|S^+|} \sum_{x \in S^+} H(x)
\]

### **Rectangularity**

\[
R = \frac{A_{\text{contour}}}{A_{\text{rect}}}
\]

### **Size**

Pixel-area of the segment.

## 2.2 Plantation Boundaries

Plantations are detected using a **fine-tuned YOLO model**, retaining only high-confidence detections.

---

# 3. Rule-Based Boundary Refinement

High-confidence segments are identified using rules defined in the paper.

## **Farm rules:**
- Entropy < **1.0**  
- Rectangularity > **0.67**  
- Size between **500–2000 m²**  
- Must be part of a cluster of **≥ 3** farms

## **Scrubland / non-agro rules:**
- Size **60,000–5,000,000 m²**  
- >50% non-agricultural pixels (via IndiaSAT v3 cross-check)

## **Plantation rules:**
- Size **1,000–20,000 m²**

## **Overlap resolution:**

\[
\text{Plantation} > \text{Farm} > \text{Scrubland}
\]

---

# 4. Sample Generation and Classifier Training

From refined boundaries, ~150 uniformly distributed samples per class are extracted for each 16×16 block.

For each sample, **Google’s 64-dimensional embedding vectors** for the **past three years** are collected.

A **Random Forest classifier** is trained **per AEZ**, allowing the model to adapt to regional conditions and avoid over-generalization. These models produce farm, non-agro, and plantation predictions at **10 m resolution**.

---

# 5. Integration into IndiaSAT

The AEZ-level predictions are integrated into the IndiaSAT v3 LULC pipeline:

1. Add built-up, water, barren classes.  
2. Insert classifier outputs (farm, non-agro, plantation).  
3. Split farm regions into four crop-intensity classes (8, 9, 10, 11).  
4. Split non-agro regions into forest and non-forest; non-forest becomes **scrubland (12)**.  
5. Assign plantation as **class 13**.

---

# Conclusion

This methodology enables accurate, large-scale differentiation of scrublands, farms, and plantations across India by leveraging high-resolution CV-derived boundaries and AEZ-specific classification. It also introduces India’s first dedicated nationwide plantation layer and improves the contextual relevance of LULC mapping across diverse landscapes.

---

