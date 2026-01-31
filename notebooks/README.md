## Notebooks

### `sncosmo.ipynb`

This notebook implements supernova template fitting using the `sncosmo` library. Standard Type Ia supernova templates (SALT2 and SALT3) are fit to each light curve, 
and goodness-of-fit metrics and fitted template parameters are extracted.

---

### `Bhardwaj25_inspired.ipynb`

This notebook implements an alternative classification pipeline inspired by the approach presented in Bhardwaj et al. (2025), 
it integrates supernova template-fitting diagnostics and sequence-model predictions from GRU and Transformer encoders. 
A gradient-boosted tree model is then trained on this combined feature set.
