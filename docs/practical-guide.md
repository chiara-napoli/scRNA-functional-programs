# Practical guide

This guide explains how to apply the framework described in Napoli et al. (2026) to organize transcriptional, regulatory, and metabolic programs derived from single-cell RNA sequencing.

The central idea is simple: start from one scRNA-seq dataset, derive complementary functional representations for the same cells, and use MOFA+ to organize their coordinated variation in a shared latent space.

The workflow consists of six stages:

1. Construct the functional views
2. Learn a shared latent representation with MOFA+
3. Build a post-modelling neighbourhood graph
4. Identify and assess functional states
5. Characterize each state within the individual views
6. Integrate the biological interpretation

> **Important:** the regulatory and metabolic views are computationally inferred from the same transcriptomic measurement. They should therefore be interpreted as complementary functional representations, not as independently measured molecular modalities.

---

## 1. Construct the functional views

Start from a quality-controlled and normalized scRNA-seq expression matrix.

For every cell, construct four representations:

| View | What it represents | Method used in the study |
|---|---|---|
| RNA | Gene-expression state | Highly variable genes |
| Regulatory | TF regulon activity | pySCENIC |
| Metabolite | Predicted metabolite-level features | scFEA |
| Flux | Predicted metabolic reaction activity | scFEA |

The four matrices contain different types and numbers of features, but they must contain the **same cells in the same order**.

Conceptually:

```text
                         ┌─ Gene expression
                         │
scRNA-seq expression ────├─ TF regulon activity
                         │
                         ├─ Metabolite-level features
                         │
                         └─ Reaction fluxes
```

### RNA view

The RNA view represents transcriptional variation using highly variable genes (HVGs).

In the published breast cancer case study, 3,386 HVGs were selected with Scanpy using:

```text
min_mean = 0.0125
max_mean = 3
min_disp = 0.5
```

These values describe the published analysis and are not intended as universal parameter settings.

### Regulatory view

Regulatory activity was inferred with **pySCENIC v0.12.0**.

The analysis consisted of three steps:

1. **GRNBoost2** to infer gene regulatory relationships;
2. **RcisTarget** to identify motif-supported regulons;
3. **AUCell** to estimate regulon activity in each cell.

This produced 79 TF-regulon activity scores per cell in the published dataset.

Unlike the RNA view, which was restricted to HVGs, regulatory inference was performed using the broader set of expressed genes.

### Metabolite and flux views

Metabolic activity was inferred with **scFEA**.

scFEA produced two separate cell-level representations:

- predicted metabolite-level features;
- predicted reaction-level metabolic fluxes.

The published analysis used the M171 human metabolic map, containing 168 reactions, 22 supermodules, and 70 intermediate metabolites, and scFEA was trained for 100 epochs.

As for regulatory inference, the broader expressed-gene space was used rather than only the HVGs selected for the RNA view.

### Output of Step 1

At the end of this stage, you should have four matrices:

```text
RNA expression        cells × genes
TF activity           cells × regulons
Metabolite features   cells × metabolites
Reaction fluxes       cells × reactions
```

Before continuing, verify that cell identifiers and cell ordering are identical across all four matrices.

---

## 2. Learn a shared latent representation

The four views have different numerical scales and therefore need to be prepared before joint modelling.

### Standardize the views

In the published analysis, each feature was **z-score scaled across cells** before MOFA+ training.

This produces comparable numerical scales for latent-factor modelling.

The standardized matrices are used only for this modelling step. Keep the original values of each view, because they will be needed later for feature-level statistical analyses.

The workflow is therefore:

```text
four aligned views
        ↓
z-score each feature across cells
        ↓
       MOFA+
        ↓
shared latent representation
```

### Fit MOFA+

The standardized views are jointly modelled with **MOFA+**.

The published analysis used:

- MOFA+ version `1.16.0`;
- Gaussian likelihoods for all four views;
- 10 latent factors.

The 10-factor solution was selected after examining model convergence and the variance explained by additional factors.

The number of factors is therefore a **model-selection decision**, not a fixed parameter of the framework.

### What does MOFA+ provide?

For every cell, MOFA+ produces coordinates in a shared latent space.

It also provides view-specific feature loadings, which indicate how strongly individual genes, regulons, metabolite-level features, and reactions contribute to the latent factors.

The latent space can therefore be used to organize cells according to coordinated variation across the four functional representations.

At the end of this stage, retain:

- the cell-level MOFA+ factors;
- the feature loadings for each view;
- the original non-standardized matrices.

---

## 3. Build the post-modelling neighbourhood graph

The MOFA+ factors describe the global latent structure of the cells.

For downstream clustering, the published analysis used these factors to construct a neighbourhood graph.

The MOFA+ factors were stored in an AnnData object as:

```text
X_mofa
```

**BBKNN** was then applied to this representation using `cell_line` as the grouping variable.

The sequence is:

```text
MOFA+ factors
      ↓
   X_mofa
      ↓
BBKNN neighbourhood graph
      ↓
downstream clustering
```

BBKNN operates on the neighbourhood graph rather than changing the MOFA+ latent coordinates themselves.

In the published dataset, this step was used to balance local neighbourhood composition across cell lines.

### Is BBKNN always required?

No.

The use of BBKNN and the choice of `cell_line` as the grouping variable were specific to the experimental design of the published case study.

For a different dataset, post-modelling harmonization should be considered only when justified by the study design and by technical or sample-associated structure in the data.

---

## 4. Identify and assess functional states

Once the neighbourhood graph has been constructed, cells can be clustered to identify candidate functional states.

The published analysis used **Leiden clustering** on the BBKNN graph.

### Choose the clustering resolution

Leiden resolutions between `0.4` and `1.0` were explored.

A resolution of `0.6` was retained for the published case study.

This value should not be treated as a universal setting. The appropriate resolution depends on the structure and biological granularity of the dataset being analysed.

### Assess cluster stability

The stability of the resulting clusters was evaluated through repeated subsampling.

For each iteration:

1. randomly retain 80% of the cells;
2. reconstruct the neighbourhood graph;
3. repeat Leiden clustering;
4. compare the new labels with the corresponding labels from the full dataset using the **Adjusted Rand Index (ARI)**.

In the published analysis, this resulted in a mean ARI of approximately:

```text
0.59 ± 0.06
```

The purpose of this procedure is not to establish a universal ARI threshold. It provides an empirical way to assess how stable the identified partition is when the dataset is perturbed.

At this point, the clusters can be regarded as candidate functional states.

The next question is: **what makes each state biologically different from the others?**

---

## 5. Characterize each functional state

To characterize the clusters, return to the individual views and identify the features associated with each state.

This is an important distinction:

```text
MOFA+ latent space  →  organize and cluster cells

individual views    →  characterize and interpret clusters
```

The statistical analyses are performed on the **original values of each view**, not on the z-score-scaled values used to train MOFA+.

In the published analysis, each cluster was compared with all remaining cells using the **Wilcoxon rank-sum test**, followed by Benjamini–Hochberg correction for multiple testing.

### RNA

Differential gene expression was calculated using log-normalized RNA expression.

Genes were considered cluster-associated when they satisfied:

```text
adjusted P < 0.05
logFC > 0.5
```

### TF regulon activity

Regulatory differences were evaluated using the original **AUCell scores**.

Because AUCell values represent regulon activity rather than gene expression, the effect size was summarized as the difference in mean activity between the target cluster and the remaining cells.

### Metabolite-level features

Predicted metabolite-level features were analysed using the original, non-z-scored scFEA outputs.

Differences between the target cluster and the remaining cells were summarized using differences in mean predicted values.

### Reaction fluxes

Predicted reaction fluxes were analysed using the original scFEA flux estimates.

As for metabolite-level features, differences between clusters were summarized using mean differences rather than gene-expression log-fold changes.

### Why use view-specific statistics?

Genes, AUCell scores, metabolite-level predictions, and reaction fluxes represent different types of quantities.

They should therefore be interpreted on their appropriate native scales rather than forcing all four views into the same feature-level statistic.

The latent model brings the views together to organize the cells; biological characterization then returns to the meaning of each individual view.

---

## 6. Integrate the biological interpretation

The final step is to connect the cluster-associated features from the different views to biological processes.

Directly comparing genes, TF activity scores, metabolites, and reaction fluxes is difficult because they represent different biological entities.

The framework therefore moves from **feature-level signals to functional annotations**.

### RNA programs

Cluster-associated genes can be tested for functional enrichment to identify biological processes associated with each state.

In the published analysis, enrichment was performed using **GSEApy/Enrichr** with Gene Ontology Biological Process annotations.

### Regulatory programs

TF regulons can be interpreted through their target genes.

The targets associated with cluster-specific regulatory activity can therefore be mapped to enriched biological processes, allowing regulatory signals to be interpreted in a pathway-level context.

### Metabolic programs

Metabolite-level features and reaction fluxes do not directly correspond to genes.

For interpretation, the published analysis linked these metabolic features to the genes associated with the corresponding reactions in the scFEA metabolic model.

These reaction-associated gene sets were then used for functional enrichment.

### Bring the evidence together

After enrichment, the different views can be interpreted within a shared biological annotation space:

```text
RNA markers
     ↓
transcriptional processes

TF regulon activity
     ↓
regulatory programs

metabolite features and reaction fluxes
     ↓
metabolic programs

     ↓

integrated interpretation
of the functional state
```

This allows each cluster to be characterized by the transcriptional, regulatory, and metabolic programs associated with it.

Signals that converge on the same biological processes across different views provide **concordant evidence from complementary transcriptome-derived representations**. View-specific signals are also informative, because regulatory or metabolic projections may highlight functional structure that is less apparent from gene expression alone.

---

## General workflow

The complete strategy can be summarized as:

```text
scRNA-seq
   │
   ├── RNA expression
   ├── TF regulon activity
   ├── metabolite-level features
   └── reaction fluxes
              │
              ↓
        standardization
              │
              ↓
            MOFA+
              │
              ↓
        latent factors
              │
              ↓
    neighbourhood graph
              │
              ↓
           Leiden
              │
              ↓
       functional states
              │
              ↓
  view-specific characterization
              │
              ↓
 integrated biological interpretation
```

The framework therefore provides a structured strategy for **organizing and interpreting complementary functional projections inferred from scRNA-seq**.

---

## Case-study parameters versus framework decisions

Several numerical settings reported above reproduce the choices made in the published breast cancer analysis. They are included to make the workflow transparent, not to prescribe fixed values for other datasets.

In particular, the following should be selected according to the dataset and analytical question:

- highly variable gene selection;
- number of MOFA+ factors;
- whether post-modelling harmonization is needed;
- the grouping variable used for harmonization;
- Leiden resolution;
- thresholds used for cluster-specific feature selection.

The general framework is defined by the analytical sequence and by the relationship between the functional views, rather than by a fixed set of parameter values.
