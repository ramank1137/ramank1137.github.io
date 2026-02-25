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

We generated pan-India LULC v4 layers from 2017-2018 to 2023-2024 which can be checked on our application. [Link to application](https://raman-461708.projects.earthengine.app/view/pan-india-lulc-indiasat-v4). Our open source code is available on this link. [Link to code](https://github.com/ramank1137/Scrubland-Field-Delineation) To report an issue with our LULC. [Link to report issue](https://github.com/ramank1137/Scrubland-Field-Delineation/issues/new?template=lulc_refinement_request.yml)
><div style="background:white; padding:10px; display:block; text-align:center;"> <img src="/assets/img/Google-app.png" alt="Pipeline" style="max-width:100%; height:auto;"> </div>
>Figure shows our application which can be accessed from the above link.

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
>Figure showing representative tile selection across India. a) India divided into agro-ecological zones (AEZs) b) Next for each AEZ representative tiles are selected. Here we show for AEZ 7. Below the grids, clusters are shown whic hare generated from Google embeddings v1 using k-means clustering (k=32). Then the tiles are selected in orange using Jensen-Shannon (JS) divergence. c) Similary representative tiles are selected from all of India which will capture all the nuances of different landscapes across all the AEZs. 

# 2. High-Resolution Boundary Delineation

High-resolution imagery (zoom 17) is downloaded for each selected block.

Two CV pipelines operate on these tiles:

## 2.1 Farm & Non-Agricultural Boundaries

We use the approach by wang et al. who uses **FracTAL-ResUNet** model to generate:

- field extent  
- boundary probability  
- distance-to-boundary

><div style="background:white; padding:10px; display:block; text-align:center;"> <img src="/assets/img/Fractal Res-Unet.png" alt="Pipeline" style="max-width:100%; height:auto;"> </div>
> The figure illustrates how FracTAL-ResUNet processes 1.19 m high-resolution satellite imagery to delineate boundaries in mixed landscapes containing both farmlands and scrublands. In the subsequent step, we apply an information-theoretic selection criterion to retain only those farm and scrubland boundaries with very high confidence.

These outputs are merged and processed by **hierarchical watershed segmentation** to obtain closed polygons.

For delineation of farm and scrubland boundaries, I extended the approach of Wang et al. (2022), which uses high-resolution imagery for field extraction but does not separate fields from adjacent scrublands. We generalized this method to mixed landscapes by leveraging the insight that unlike naturally shaped scrublands, agricultural fields exhibit lower geometric and textural randomness which can be captured via entropy, rectangularity, and segment-size
metrics. Now to seperate out the boundaries of farms and non-agro regions which will include scrublands we compute the following for each segement which will be used in the following module to seperate out farms and non-agro from the boundaries generated:

#### Entropy

$$
H = - \sum_{i=1}^{L} p_i \log_2(p_i)
$$

Segment-level entropy:

$$
\overline{H}_S
= \frac{1}{|S^+|}
\sum_{x \in S^+} H(x)
$$

#### Rectangularity

$$
R = \frac{A_{\text{contour}}}{A_{\text{rect}}}
$$

#### Size  
Computed as pixel-area of the segment.

><div style="background:white; padding:10px; display:block; text-align:center;"> <img src="/assets/img/Segments.png" alt="Pipeline" style="width:60%; height:auto;"> </div>
> The figure shows segments coresponding to farm, scrubland and plantations. A) A farm will generally exhibit very less entropy due to its smooth textural appearance on the rgb image which can be captured using entropy. Farms are generally rectangular in shape which is captured by rectangularity. In india most farms are small and hence area can be used to filter them out. B) A scrubland will high entropy due to its rugged textural appearance which is due to its natural formation. This can be captured with high entropy. Fractal-ResUnet usually segments them as irregular boundaries as seen in the figure which will give lower values for rectangularity. C) Plantations may appear as of rectangular shape but will exhibit higher entropy due to its gridded plantation pattern. So they will be eliminated here but will be capture by our Yolo model.
> The figure illustrates segments corresponding to farms, scrublands, and plantations: A) Farms. Farms typically exhibit low entropy because of their smooth textural appearance in RGB imagery. Their rectangular geometry is well captured by rectangularity, and in India, most farms are relatively small in area, so can be filtered well through size. B) Scrublands. Scrublands display high entropy due to their naturally rugged and heterogeneous texture. FracTAL-ResUNet often delineates them with irregular boundaries, resulting in low rectangularity scores. Size plays an important role for them as generally the patches are of very large size. C) Plantations. Plantations may appear rectangular in shape, but their gridded planting patterns yield higher entropy values. As a result, they are filtered out in this stage; however, they are later captured by our YOLO model, which detects plantation-specific structural cues. 

## 2.2 Plantation Boundaries

For plantations, we fine-tuned a YOLO detection model to identify plantations across heterogeneous landscapes at the same spatial resolution. To train this model, we curated sparse ground truth from multiple regions and expanded it using augmentation techniques like cut-mix.

><div style="background:white; padding:10px; display:block; text-align:center;"> <img src="/assets/img/plantations_ARCH.jpg" alt="Pipeline" style="max-width:100%; height:auto;"> </div>
> The figure illustrates the training and inference workflow of the YOLO model trained on our plantation dataset. To improve robustness, we apply CutMix-based augmentation, grafting plantation patches onto tiles that either lack plantations or contain plantation-like gridded patterns. This reduces false positives and helps the model better discriminate true plantation structures.

# 3. Rule-Based Boundary Refinement

To refine the boundaries and filter out high-confidence segments for farms, non-agro(including scrubs) and plantation we identified follwing rules.

## Farm rules

- >Entropy < 1.0  
- >Rectangularity > 0.67  
- >Size ∈ [500, 2000] m²  
- >Must belong to a cluster of ≥ 3 farms

## Scrubland / non-agro rules

- >Size ∈ [60,000, 5,000,000] m²  
- >50% non-agricultural pixels (via IndiaSAT v3)

## Plantation rules

- >Size ∈ [1,000, 20,000] m²

><div style="background:white; padding:10px; display:block; text-align:center;"> <img src="/assets/img/Seperating Boundaries.png" alt="Pipeline" style="width:50%; height:auto;"> </div>
> The figure depicts the selection of high-confidence boundaries for farms, scrublands, and plantations from the full set of detected boundaries. These selected boundaries serve as the basis for sample extraction. Since only a small number of samples are required from each tile, we retain just a few high-confidence boundaries per tile. The remaining large number of segments falls within the grey region.

## Overlap hierarchy

$$
\text{Plantation} > \text{Farm} > \text{Scrubland}
$$

# 4. Sample Generation and Classifier Training

Once we have the high-quality segements, we then generated samples from each of these grids uniformly. For each sample, **Google’s 64-dimensional embeddings** for the **past 3 years** are collected. Using these samples we trained **a pool of classifiers for each AEZ**, allowing regional adaptation and avoiding over-generalization. The resulting models produce farm, non-agro, and plantation predictions at **10 m spatial resolution**.

><div style="background:white; padding:10px; display:block; text-align:center;"> <img src="/assets/img/Sample.png" alt="Pipeline" style="max-width:100%; height:auto;"> </div>
> The figure shows pan-India sample locations, with farm, scrubland, and plantation samples displayed as black dots.

# 5. Integration into IndiaSAT

The AEZ-level outputs are merged with the IndiaSAT v3 framework:

1. Add built-up, water, and barren classes  
2. Insert farm, non-agro, and plantation predictions  
3. Split farms into crop-intensity classes (8, 9, 10, 11)  
4. Split non-agro into forest vs. non-forest; non-forest is scrubland (12)  
5. Assign plantations as class (13)

# Conclusion

This methodology enables accurate mapping of scrublands, farms, and plantations across India by combining high-resolution CV-derived boundaries with AEZ-specific modeling. It produces India’s first nationwide plantation layer and significantly improves distinction between scrublands and rain-fed agriculture.

