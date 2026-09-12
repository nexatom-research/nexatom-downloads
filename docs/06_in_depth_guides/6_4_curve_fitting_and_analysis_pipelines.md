## Curve Fitting and Analysis Pipelines

To minimize computational latency and language barrier overhead, the NexatomTT SDK performs intensive mathematical fitting directly within the native C++ library. The SDK utilizes optimized, multi-threaded Levenberg-Marquardt non-linear least squares solvers.

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

> **Auto-Select Mode:** If configured to `AUTO`, the SDK evaluates the Single, Bi-Exponential, and Gaussian models simultaneously, returning the parameters of the model yielding the lowest reduced chi-squared ($\chi_\nu^2$) statistic.

### [Correlation fitting (Multi-Tau CORM)](6_4_curve_fitting_and_analysis_pipelines.md#correlation-fitting)

For Intensity Correlation ($g^{(2)}(\tau)$) evaluated on a quasi-logarithmic lag scale, the SDK computes comprehensive goodness-of-fit metrics alongside the primary decay parameters.

The solver yields the **Correlation Time** ($\tau_c$) and the **Coherence Factor** ($\beta$). Furthermore, the SDK natively computes:
*   $\chi^2$ and Reduced $\chi^2$ ($\chi_\nu^2$)
*   Coefficient of Determination ($R^2$)
*   Signal-to-Noise Ratio (SNR)

To reject baseline noise at extreme lag times, users can restrict the solver via ROI-constrained fitting. The fitted curve is evaluated at the original discrete lag times and appended to the callback payload.

### [Dynamic Light Scattering (DLS) analysis](6_4_curve_fitting_and_analysis_pipelines.md#dynamic-light-scattering-analysis)

When DLS analysis is enabled (`nexatom_tt_enable_dls_analysis()`), the SDK applies Cumulant Analysis to the correlation data to extract nanoparticle sizing metrics.

By defining the experimental wavelength, scattering angle, temperature, and solvent viscosity, the SDK directly computes the diffusion coefficient. The output `nexatom_dls_analysis_result_t` struct yields:
*   **Z-average diameter:** Intensity-weighted mean hydrodynamic size.
*   **Polydispersity Index (PDI):** Dimensionless measure of the broadness of the size distribution.
*   **Mean decay rate ($\Gamma$)**, Variance, and Skewness.

### [Fluorescence Correlation Spectroscopy (FCS) analysis](6_4_curve_fitting_and_analysis_pipelines.md#fluorescene-correlation-spectroscopy-analysis)

The FCS module decomposes concentration and diffusion kinetics from confocal optical setups. Configuration requires strict definition of the confocal volume (lateral and axial waist radii).

The solver accommodates multiple complex physical phenomena:
*   **Multi-component diffusion:** Resolves single and two-component kinetic models.
*   **Triplet state correction:** Compensates for fluorophores entering dark states.
*   **Anomalous diffusion:** Fits the $\alpha$ parameter for non-Brownian sub-diffusion in crowded environments (e.g., live cells).
*   **Flow component separation:** Resolves directed active transport versus passive thermal diffusion.

Outputs include absolute concentration, particles per volume, and characteristic diffusion times.

### [Diffuse Correlation Spectroscopy (DCS) analysis](6_4_curve_fitting_and_analysis_pipelines.md#diffuse-correlation-spectroscopy-analysis)

Designed for deep-tissue in-vivo hemodynamics, the DCS module fits the semi-infinite photon diffusion equation to the measured temporal auto-correlation of scattered light.

Users must provide tissue optical properties (absorption coefficient $\mu_a$, reduced scattering coefficient $\mu_s'$) and source-detector separation. The SDK outputs:
*   **Blood Flow Index (BFI):** Relative index of microvascular perfusion.
*   **Decorrelation time:** Inverse indicator of scatterer movement speed.
*   **Brownian motion parameters:** To model erythrocyte displacement.
