# 6.4 Curve fitting and analysis pipelines

TIHI fitting and correlation-derived DLS/FCS/DCS analysis run on the host. Configuration selects models, fit ranges and experimental parameters; successful configuration does not guarantee a converged or physically meaningful fit.

## TIHI

Configure acquisition first, then optional background subtraction, signal/background regions, model, minimum counts, convergence threshold and iteration limit. Read the fitting record's attempted/converged/curve flags and valid bins. Preserve the original histogram and background settings; a fitted lifetime requires an appropriate experiment and model.

## Correlation and physical analyses

CORL/CORM return lag axes and normalization statistics. Check `normalization_valid`; values without valid normalization must not be labelled normalized g². DLS/FCS/DCS outputs are embedded in the CORM callback, not delivered through standalone analysis callback setters.

The C ABI exports one `analysis_result` selected by `analysis_type`: NONE, DLS, FCS or DCS. If multiple results are available, selection priority is DLS, then FCS, then DCS. Enable only the analysis intended for the exported result where practical.

| Analysis | Inputs to establish before interpretation |
| --- | --- |
| DLS | Wavelength, scattering angle, temperature, viscosity and refractive index |
| FCS | Confocal dimensions, experimental conditions/calibration diffusion and selected fit model |
| DCS | Optical tissue parameters, source/detector separation, model parameters and wavelength |

Use `fit_result` for quality, fitted curve, ROI, beta and signal-to-noise. Reserved quality fields inside nested analysis records currently remain zero. Uncomputed derived values can be NaN/zero; do not turn them into successful measurements. For example, a record field named perfusion or flow is not by itself a validated clinical measurement.

The advanced [DLS](../07_c_api/7_12_dls.md), [FCS](../07_c_api/7_13_fcs.md) and [DCS](../07_c_api/7_14_dcs.md) reference pages retain the relevant controls. Product application notes remain separate from this SDK manual.

[In-depth guides](index.md)
