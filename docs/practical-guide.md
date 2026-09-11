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
