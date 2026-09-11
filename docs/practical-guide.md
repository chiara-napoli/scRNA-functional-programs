# Practical guide

This guide provides a practical description of the workflow presented in
Napoli et al. (2026) for organizing regulatory and metabolic programs inferred
from single-cell RNA sequencing.

The framework derives multiple functional representations from the same
scRNA-seq measurement and organizes their coordinated variation within a
shared latent space.

It is intended as a methodological companion to the published article rather
than as a replacement for the original Methods section.

> **Note**
> The regulatory and metabolic views described here are computationally
> inferred from transcriptomic data. They are not independently measured
> molecular modalities.

---

## Workflow at a glance

The framework consists of six main stages:

1. Data collection and construction of functional views
2. Multi-view latent factor modelling
3. Post-modelling harmonization
4. Clustering
5. Layer-specific feature selection
6. Integrative biological interpretation

---

## 1. Construct the functional views

Start from a quality-controlled and normalized scRNA-seq expression matrix.

Four aligned representations are constructed for each cell:

| View | Representation | Method used in the study |
|---|---|---|
| RNA | Gene-expression features | Highly variable genes |
| Regulatory | TF regulon activity | pySCENIC |
| Metabolic | Predicted metabolite-level features | scFEA |
| Metabolic | Predicted reaction-level fluxes | scFEA |

All four matrices must represent the **same cells in the same order** to
maintain one-to-one correspondence across views.

### RNA view

For the RNA representation, highly variable genes can be selected from the
normalized expression matrix.

In the published breast cancer case study, 3,386 HVGs were selected with
Scanpy using:

- `min_mean = 0.0125`
- `max_mean = 3`
- `min_disp = 0.5`

These values describe the published case study and should not be interpreted
as universal parameter recommendations.

### Regulatory view

TF regulon activity was inferred with **pySCENIC v0.12.0** using its standard
three-stage workflow:

1. GRN inference with **GRNBoost2**
2. Motif enrichment with **RcisTarget**
3. Regulon activity scoring with **AUCell**

In the published analysis, this resulted in activity scores for 79 TF
regulons.

Importantly, functional inference was performed using the full set of
expressed genes rather than only the HVGs used for the RNA representation.

### Metabolic views

Metabolic representations were inferred with **scFEA**, producing two
distinct views:

- predicted metabolite-level features;
- predicted reaction-level metabolic fluxes.

In the published analysis, scFEA used the M171 human metabolic map,
comprising 168 reactions, 22 supermodules, and 70 intermediate metabolites.

The model was trained for 100 epochs to obtain cell-level metabolic
predictions.

As with the regulatory view, metabolic inference was performed from the
broader expressed-gene space rather than the HVG-restricted RNA view.

---

### Before moving to latent-factor modelling

Check that:

- every view contains the same cells;
- cell identifiers and ordering are consistent across matrices;
- each matrix has cells as observations and its corresponding biological
  entities as features.

The views represent different feature spaces — genes, TF regulons,
metabolites, and metabolic reactions — and do not need to contain matching
features.

---

## 2. Prepare the views and learn the latent representation

Once the four functional views have been constructed and aligned, they can be prepared for joint latent-factor modelling.

### Standardize the views

Before MOFA+ training, each view should be standardized across cells.

In the published analysis, all four views were **z-score scaled across cells** to place features on a comparable scale before latent-factor modelling.

This standardization is used for model fitting. It is important to retain the original, non-standardized values of each inferred view for downstream layer-specific analyses.

Conceptually, the input consists of four aligned matrices:

```text
RNA expression        cells × genes
TF activity           cells × regulons
Metabolite features   cells × metabolites
Reaction fluxes       cells × reactions
```

The feature spaces are different, but the cell dimension is shared across all views.

### Multi-view modelling with MOFA+

The standardized matrices are jointly modelled using **MOFA+**.

In the published study:

- MOFA+ version `1.16.0` was used;
- the four views were modelled with Gaussian likelihoods;
- a model with 10 latent factors was retained.

The number of factors should be treated as a model-selection choice rather than a fixed property of the framework. In the published case study, 10 factors were selected after assessing model convergence and the variance explained across factors, with diminishing returns from additional factors.

MOFA+ produces a shared low-dimensional representation of the cells while retaining view-specific feature loadings.

### How to interpret the latent space

In this framework, MOFA+ is used to **organize complementary functional projections of the same transcriptomic measurement**.

The latent factors should therefore not be interpreted as integrating four independently measured molecular modalities or as increasing the amount of molecular information available from the experiment.

Instead, they provide a structured representation of coordinated transcriptional, regulatory, and metabolic variation inferred from the same cells.

Feature loadings can then be inspected across views to understand which genes, regulons, metabolites, and metabolic reactions contribute to each latent factor.

### Before moving downstream

At this stage, retain:

- the MOFA+ latent factors for each cell;
- the view-specific feature loadings;
- the original non-z-scored values for downstream feature-level analyses.

The latent factors provide the cell representation used in the next stage of the workflow.
