# MALLORN TDE Classification
https://www.kaggle.com/competitions/mallorn-astronomical-classification-challenge
## Overview

This project focuses on identifying **tidal disruption events (TDEs)** from multiband photometric light curves in the MALLORN (LSST-style) Kaggle competition. 
TDEs are rare transients produced when stars are disrupted by supermassive black holes and are observationally difficult to distinguish from supernovae and AGN using photometry alone.

The main challenge is not detecting variability, but **discriminating TDEs from more common transients** under strong class imbalance, irregular cadence, and incomplete spectral information.
<img src="tde_lightcurve_flux" width="600">
---

## Approach

I built a **hybrid classification pipeline** that combines physics-informed feature engineering, explicit supernova template fitting,
sequence models operating directly on light curves, and a tree-based ensemble to integrate all signals.

The guiding idea was to encode the physics of TDEs and transients (color evolution, decay behavior, template consistency) 
and let machine learning combine these perspectives.

---

## Physics-Informed Feature Engineering

A central part of the project was extracting features that reflect known physical differences between TDEs and supernovae.
I implemented Gaussian-process–based features inspired by recent literature, fitting light curves jointly in time and wavelength. 
From these fits, I derived quantities describing rise and decay timescales, characteristic amplitudes, colors, and color evolution. 
These features capture an important physical distinction: **TDEs tend to remain blue, while supernovae redden over time**.

To further probe post-peak behavior, I added decay-focused features that quantify fading rates across multiple timescales and photometric bands. These features were computed in the 
rest frame to account for cosmological time dilation. This treatment is not standard in existing work, and its effect on classification performance was not clearly separable from other modeling choices. 
Somewhat surprisingly, incorporating redshift information did not appear to have a large impact, 
suggesting that this aspect is either subdominant in current models or absorbed indirectly by other features. This is a natural area for future investigation.

---

## Supernova Template Fitting

## Supernova Template Fitting

A major source of false positives in TDE searches is contamination from supernovae, which can produce light curves that look very similar when only photometric data are available. 
Type Ia supernovae in particular follow relatively well-understood and repeatable brightness patterns over time, making them a useful reference point.

To take advantage of this, I fit standard Type Ia supernova light-curve templates (SALT2 and SALT3) to every object using the `sncosmo` library. 
These templates are physically motivated models that describe how a typical supernova brightens, fades, and changes color, with a small number of parameters controlling the overall shape and color evolution. 
The `sncosmo` library provides tools to fit these models to observational data and to quantify how well they match a given light curve.

Rather than using template fitting as a hard classifier, I treated it as a diagnostic step. From each fit, I extracted goodness-of-fit metrics and fitted template parameters and
passed them to the final machine-learning model as features. Conceptually, this allows the model to learn how “supernova-like” an event is.

This provides a strong negative prior: objects that are well explained by a standard supernova model are unlikely to be tidal disruption events, even if their light curves appear unusual at first glance. 
In practice, these features were effective at reducing confusion between TDEs and supernovae and helped the classifier focus on genuinely non-supernova behavior.


---

## Sequence Models on Raw Light Curves

In parallel, I trained sequence models directly on the multiband light curves to learn temporal patterns without explicit physical assumptions.
I used both a GRU-based encoder and a lightweight Transformer encoder, operating on per-epoch representations that include flux, uncertainty, cadence information, and detection context. To improve robustness to real survey conditions, 
I applied data augmentation such as band dropout (biased toward blue bands) and cadence dropout.
Instead of using these models as standalone classifiers, I treated their out-of-fold predictions as **learned high-level summaries** of each light curve and passed them as features to the final ensemble.

---
## Gaussian-Process Modeling of Light Curves

A central component of the feature engineering pipeline is the use of **Gaussian Processes (GPs)** to model light curves in a smooth, physically interpretable way. 
Photometric observations are often sparse, irregularly sampled, and noisy, which makes it difficult to directly measure quantities such as peak brightness, decay rates, or color evolution from raw data alone.

Gaussian Processes provide a flexible way to interpolate light curves while accounting for measurement uncertainties. 
For each object, I fit independent GPs to the light curves in individual photometric bands, using a Matérn kernel with an explicit noise term. 
These fits produce smooth, continuous representations of the underlying flux evolution that can be evaluated on a common time grid.

Following this, I constructed a pseudo-bolometric light curve by summing the GP-predicted flux across bands. 
This allows physically motivated quantities, such as peak brightness, decay slopes, and time spent above a given fraction of peak flux, to be defined consistently even when observations are incomplete or unevenly sampled.

All time-based features were computed in the **rest frame**, aligning events relative to their inferred peak and correcting for cosmological time dilation using redshift information. 
This ensures that decay rates and timescales are comparable across objects at different distances.

---

## GP-Derived Features and Physical Motivation

From the GP reconstructions, I extracted several families of features designed to capture known differences between TDEs and supernovae:

- **Bolometric decay slopes** measured over multiple post-peak windows, motivated by the fact that TDEs typically fade more slowly than supernovae.
- **Per-band decay rates**, which probe whether fading behavior is consistent across wavelengths.
- **Time-above-threshold features**, measuring how long an object remains bright after peak.
- **Amplitude ratios**, comparing peak brightness to baseline flux levels.
- **Color and color-evolution features** (e.g. g–r at peak and its change over time).

Some of these features became important for the tree models.

---

## GP-Derived Features and Physical Motivation

From the GP reconstructions, I extracted several families of features designed to capture known differences between TDEs and supernovae:

- **Bolometric decay slopes** measured over multiple post-peak windows, motivated by the fact that TDEs typically fade more slowly than supernovae.
- **Per-band decay rates**, which probe whether fading behavior is consistent across wavelengths.
- **Time-above-threshold features**, measuring how long an object remains bright after peak.
- **Amplitude ratios**, comparing peak brightness to baseline flux levels.
- **Color and color-evolution features** (e.g. g–r at peak and its change over time).

Several of these features proved informative and were assigned significant weight by the tree-based models.

---
## Improving on Bhardwaj et al. (2025)  
https://arxiv.org/abs/2509.25902

Bhardwaj et al. (2025) present a strong, recent approach to photometric transient classification based on Gaussian-process modeling and physically motivated summary statistics.
While their work is developed and evaluated on a different dataset, it provides a useful starting point.

Inspired by this framework, I adapted the approach by integrating explicit supernova template fitting, additional GP-derived decay and color features,
and sequence-model predictions from GRU and Transformer encoders. An XGBoost model trained on these features produced comparable results.





---

## Practical Considerations

GP fitting is computationally expensive at scale, so I implemented a caching system that stores extracted GP features to disk. This allows features to be computed once and reused across experiments, reducing iteration time from minutes to seconds during model development.

Overall, Gaussian-process modeling served as a bridge between raw observational data and higher-level physical intuition, providing stable and interpretable features that complemented both deep learning and template-based approaches.

---

## Model Integration

All feature families were combined using gradient-boosted decision trees. 
Tree-based models were chosen for their ability to handle heterogeneous feature types, non-linear interactions, and missing or noisy inputs. 
I experimented with multiple tree model families and found that shallow, well-regularized trees were most effective. 

The GRU and Transformer models produce probabilistic outputs, but these raw probabilities were not always well calibrated. 
To address this, I applied **Platt scaling** to their out-of-fold predictions before passing them to the tree ensemble. Platt scaling is a post-processing calibration method 
that fits a simple logistic model to map raw model scores to better-calibrated probabilities.

Calibrating the sequence-model outputs made them easier for the tree ensemble to interpret and combine with other features. 
This reduced sensitivity to overconfident predictions from individual models and improved the stability of the final classifier.



## Tools and Libraries

- Python, PyTorch  
- CatBoost, XGBoost, LightGBM  
- Gaussian Processes (`george`)  
- Supernova modeling (`sncosmo`, SALT2/SALT3)  
- Milky Way extinction correction (`extinction`)  
- Stratified cross-validation with out-of-fold prediction handling
