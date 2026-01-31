## Notebooks

### `sncosmo.ipynb`

This notebook implements supernova template fitting using the `sncosmo` library. Standard Type Ia supernova templates (SALT2 and SALT3) are fit to each light curve, 
and goodness-of-fit metrics and fitted template parameters are extracted.

---

### `Bhardwaj25_inspired.ipynb`

This notebook implements an alternative classification pipeline inspired by the approach presented in Bhardwaj et al. (2025), 
it integrates supernova template-fitting diagnostics and sequence-model predictions from GRU and Transformer encoders. 
A gradient-boosted tree model is then trained on this combined feature set.

### `tde-classification-final.ipynb`

This notebook contains the initial large-scale end-to-end classification pipeline developed for the competition. It serves as the main experimental workspace where the full modeling approach was first assembled and evaluated.
This is a clean final version.

The notebook includes training of sequence models (GRU and Transformer encoders) directly on multiband light curves, Gaussian-process modeling for feature extraction, and integration of these components into a tree-based classifier. 
It also covers feature engineering, cross-validation setup, model calibration, and Platt scaling.


