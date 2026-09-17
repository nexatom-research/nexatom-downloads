## Curve Fitting and Analysis Pipelines

The NexatomTT SDK performs mathematical fitting inside the native host library so applications can receive both measured data and derived results through the same API. Its fitting routines use nonlinear least-squares methods, including Levenberg-Marquardt. These are host computations, separate from the FPGA's histogram and correlation engines. A converged numerical fit still requires an experiment consistent with the model assumptions.

### [TIHI lifetime fitting models](6_4_curve_fitting_and_analysis_pipelines.md#tihi-lifetime-fitting-models)

When `nexatom_tt_enable_tihi_fitting()` is active, the SDK attempts to fit the accumulated Time Interval Histogram (TCSPC) data to a specified mathematical model prior to dispatching the callback. The resulting parameters are populated in the `NexatomTihiFittingData` struct.

The following models are natively supported:

**Single Exponential:** (Fluorescence lifetime, simple decay)
$$y(t) = A \cdot \exp\left(-\frac{t}{\tau}\right) + B$$

**Bi-Exponential:** (Complex decay, mixed fluorophores)
$$y(t) = A_1 \cdot \exp\left(-\frac{t}{\tau_1}\right) + A_2 \cdot \exp\left(-\frac{t}{\tau_2}\right) + B$$

**Gaussian:** (Instrument Response Function, symmetric pulse analysis)
$$y(t) = A \cdot \exp\left(-\frac{(t-\mu)^2}{2\sigma^2}\right) + B$$

**Lorentzian:** (Spectral line shapes, resonance profiles)
$$y(t) = \frac{A}{1 + \left(\frac{t-\mu}{\gamma}\right)^2} + B$$

**Stretched Exponential (Kohlrausch):** (Disordered systems, heterogeneous relaxation)
$$y(t) = A \cdot \exp\left(-\left(\frac{t}{\tau}\right)^\beta\right) + B$$

> **Auto-select mode:** `NEXATOM_FIT_AUTO` tries the exponential, Gaussian, Lorentzian, bi-exponential and stretched-exponential models. Among converged candidates, the implementation selects the highest $R^2$. This is not a complexity-penalized model-selection test; choose a physically justified model when interpreting a lifetime.

Set the histogram channels, bin width and acquisition duration before fitting. Configure background subtraction and signal/background regions, then the minimum count requirement, convergence threshold and iteration limit. The input bin width is in ps; fitted lifetimes, centers and widths use the named `_ns` fields. Inspect `attempted`, `converged`, `fitted_curve_available` and `fitted_curve_size` before using a curve. Preserve the original bins and background settings alongside the fit.

<a id="correlation-fitting"></a>

### [Correlation fitting (Multi-Tau CORM)](6_4_curve_fitting_and_analysis_pipelines.md#correlation-fitting)

For Intensity Correlation ($g^{(2)}(\tau)$) evaluated on a quasi-logarithmic lag scale, the SDK computes comprehensive goodness-of-fit metrics alongside the primary decay parameters.

The solver yields the **Correlation Time** ($\tau_c$) and the **Coherence Factor** ($\beta$). Furthermore, the SDK natively computes:
*   $\chi^2$ and Reduced $\chi^2$ ($\chi_\nu^2$)
*   Coefficient of Determination ($R^2$)
*   Signal-to-Noise Ratio (SNR)

To reject baseline noise at extreme lag times, users can restrict the solver via ROI-constrained fitting. The fitted curve is evaluated at the original discrete lag times and appended to the callback payload.

First check `normalization_valid`: without it, the returned correlation values must not be labelled normalized g². Public fit-range setters select lag-array **indices**, not times. The CORM `fit_result` contains the main quality flags, ROI, `correlation_time_ns`, `beta`, fitted curve and statistics. DLS/FCS/DCS results are embedded in the CORM callback; there are no separate analysis callback setters.

The exported `analysis_result.analysis_type` selects NONE, DLS, FCS or DCS. If more than one analysis result exists, export priority is DLS, then FCS, then DCS. Enable the intended analysis alone where practical. Reserved quality fields in nested analysis records currently remain zero; use the main `fit_result` for convergence and fit quality.

<a id="dynamic-light-scattering-analysis"></a>

### [Dynamic Light Scattering (DLS) analysis](6_4_curve_fitting_and_analysis_pipelines.md#dynamic-light-scattering-analysis)

When DLS analysis is enabled (`nexatom_tt_enable_dls_analysis()`), the SDK applies Cumulant Analysis to the correlation data to extract nanoparticle sizing metrics.

Supply wavelength in nm, scattering angle in degrees, temperature in Celsius, viscosity in mPa·s and refractive index through `set_dls_experimental_conditions`. The DLS result carries the calculated diffusion/size values and experimental context. For a suitable diffusing-particle experiment, useful quantities include:
*   **Z-average diameter:** Intensity-weighted mean hydrodynamic size.
*   **Polydispersity Index (PDI):** Dimensionless measure of the broadness of the size distribution.
*   **Mean decay rate ($\Gamma$)**, Variance, and Skewness.

Set the fitting range using array indices and configure bounds/initial values as required. A radius/diameter field being present does not establish that every derived statistic was calculated; preserve NaN/unavailable values rather than replacing them with a successful zero measurement.

<a id="fluorescene-correlation-spectroscopy-analysis"></a>

### [Fluorescence Correlation Spectroscopy (FCS) analysis](6_4_curve_fitting_and_analysis_pipelines.md#fluorescene-correlation-spectroscopy-analysis)

The FCS module decomposes concentration and diffusion kinetics from confocal optical setups. Configuration requires strict definition of the confocal volume (lateral and axial waist radii).

The underlying analysis includes models for several physical phenomena:
*   **Multi-component diffusion:** Resolves single and two-component kinetic models.
*   **Triplet state correction:** Compensates for fluorophores entering dark states.
*   **Anomalous diffusion:** Fits the $\alpha$ parameter for non-Brownian sub-diffusion in crowded environments (e.g., live cells).
*   **Flow component separation:** Resolves directed active transport versus passive thermal diffusion.

Outputs include absolute concentration, particles per volume, and characteristic diffusion times.

The public preview.8 C/Python interface exposes enable, fit-range, confocal-volume, experimental-condition and fitting-control methods. It does not expose a public selector for every internal FCS model listed above. Do not infer that a two-component, anomalous or flow fit was selected merely because the result structure reserves fields for it. Inspect the returned model/technique and validity.

`set_fcs_confocal_volume(omega_xy_um, omega_z_um)` takes micrometres, not nm. `set_fcs_experimental_conditions` takes Celsius, mPa·s, wavelength in nm and a calibration diffusion coefficient in µm²/s. Incorrect units can produce plausible-looking but physically incorrect derived results.

<a id="diffuse-correlation-spectroscopy-analysis"></a>

### [Diffuse Correlation Spectroscopy (DCS) analysis](6_4_curve_fitting_and_analysis_pipelines.md#diffuse-correlation-spectroscopy-analysis)

Designed for deep-tissue in-vivo hemodynamics, the DCS module fits the semi-infinite photon diffusion equation to the measured temporal auto-correlation of scattered light.

Users must provide tissue optical properties (absorption coefficient $\mu_a$, reduced scattering coefficient $\mu_s'$) and source-detector separation. The SDK outputs:
*   **Blood Flow Index (BFI):** Relative index of microvascular perfusion.
*   **Decorrelation time:** Inverse indicator of scatterer movement speed.
*   **Brownian motion parameters:** To model erythrocyte displacement.

Use absorption and reduced-scattering coefficients in cm⁻¹, source-detector separation in cm, and the documented anisotropy, refractive-index and wavelength settings. Uncomputed values such as derived flow/perfusion fields may remain NaN or zero. Their presence in a record is not a validated clinical measurement. Retain the correlation curve, fitting ROI and model parameters so a result can be independently interpreted.
