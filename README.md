#  Organizing regulatory and metabolic programs from scRNA-seq

### A practical companion to a latent-factor framework for functional representations derived from single-cell transcriptomics

Single-cell RNA sequencing directly measures transcript abundance, while computational methods can derive complementary representations of regulatory and metabolic cellular states.

This repository provides a practical guide to the framework presented in:

> **Napoli C, Bardozzo F, Verma S, et al. (2026)**  
> *A latent factor framework to organize regulatory and metabolic programs inferred from scRNA-seq*  
> **Bioinformatics Advances**, 6(1), vbag185.  
> DOI: [10.1093/bioadv/vbag185](https://doi.org/10.1093/bioadv/vbag185)

---

##  Framework overview

The framework starts from a single scRNA-seq measurement and derives four complementary functional representations:

- **Gene expression** - transcriptional state
- **TF regulon activity** - regulatory programs inferred with pySCENIC
- **Metabolite-level features** - metabolic features predicted with scFEA
- **Reaction fluxes** - metabolic reaction activities predicted with scFEA

These representations are standardized and jointly organized using **MOFA+**, providing a shared latent representation for organizing and interpreting coordinated regulatory, metabolic, and transcriptional programs.
<p align="center">
  <img src="figures/pipeline.png" alt="Overview of the scRNA-seq functional program framework" width="100%">
</p>

*Overview of the framework, from transcriptome-derived functional representations to latent-factor modelling, post-modelling harmonization, clustering, feature selection, and integrative biological interpretation. Framework overview reproduced from Napoli et al. (2026).*
> **Important:** the four views are derived from the same transcriptomic measurement. They should therefore be interpreted as complementary computational representations rather than independent omics measurements.
