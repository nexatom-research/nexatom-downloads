## Measurement Modules

This section documents the advanced mathematical and analytical configuration endpoints. These methods dictate how the internal C++ solvers evaluate data (e.g., background subtraction, curve fitting, and physical modeling) before dispatching the payload to your Python callbacks.

Prepare acquisition controls while output is quiet, check the native profile, then explicitly enable/start the intended engines. Analysis settings do not start hardware acquisition. Native parameter checks remain authoritative; an accepted configuration does not prove that the signal fits the chosen physical model.

<a id="time-interval-histogram-config"></a>

### [Time Interval Histogram (TIHI) advanced config](5_5_measurement_modules.md#time-interval-histogram-config)

| Method | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `set_tihi_background_method` | `method: int` | `None` | Sets the baseline subtraction mode: `NONE`, `USER_CONSTANT`, or `USER_REGION`. |
| `set_tihi_user_background_value` | `value: float` | `None` | Defines the static baseline subtracted from all bins if `USER_CONSTANT` is active. |
| `set_tihi_signal_region` | `start_bin: int`, `end_bin: int` | `None` | Restricts fitting algorithms and SNR calculations to a specific Region of Interest (ROI). |
| `set_tihi_background_region` | `start_bin: int`, `end_bin: int` | `None` | Selects background bins for `USER_REGION` subtraction. |
| `set_tihi_fitting_parameters` | `min_counts: int`, `convergence_threshold: float`, `max_iterations: int` | `None` | Sets minimum data and convergence/iteration limits. |
| `enable_tihi_fitting` | `enable: bool` | `None` | Activates the Levenberg-Marquardt non-linear least squares solver on the active histogram. |
| `set_tihi_fitting_model` | `model: int` | `None` | Selects `NEXATOM_FIT_AUTO`, `NEXATOM_FIT_EXPONENTIAL`, `NEXATOM_FIT_BI_EXPONENTIAL`, `NEXATOM_FIT_GAUSSIAN`, `NEXATOM_FIT_LORENTZIAN` or `NEXATOM_FIT_STRETCHED_EXP`. |

Configure normal TIHI with `set_time_histogram_channels(start_channel, stop_channel)`, `set_time_histogram_bin_width(bin_width_ps)`, `set_time_histogram_num_bins(num_bins)`, and choose what results cover and when the run ends with `set_result_span()` and `set_run_end()` (see [result model](5_1_nexatomtt_library.md#result-model)). Fast TIHI is a separate capability (`start_fast_tihi`, `stop_fast_tihi`, `set_fast_tihi_result_callback`), gated by the profile's feature bits, not an alternative selected from the device name. It uses the same result model with `NEXATOM_RESULT_PROCESSOR_FAST_TIME_HISTOGRAM`.

<a id="multi-fold-coincidence-config"></a>

### [Multi-Fold Coincidence (MFCO) advanced config](5_5_measurement_modules.md#multi-fold-coincidence-config)

| Method | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `set_multifold_coincidence_pattern_filter` | `requirements: iterable[int]` | `None` | Takes exactly eight `NEXATOM_MFCO_REQ_*` entries: `DONT_CARE`, `REQUIRED` or `FORBIDDEN`. This configures host pattern analysis. |
| `disable_multifold_coincidence_pattern_filter` | *None* | `None` | Resets all channel requirements to `DONT_CARE`. |
| `set_mfco_background_method` | `method: int` | `None` | Sets the noise subtraction mode: `NONE`, `USER_CONSTANT`, or `USER_SELECTED_PATTERN_BIN`. |

The acquisition channel setter accepts channel IDs for MFCO slots (`[0, 1]`, for example), with unused slots padded with `0xff`. It does not accept a bitmask or enable booleans. Pattern/mask metadata is not proof that the hardware physically gated every unselected input. Read terminal and quality flags before computing rates from returned bins.

<a id="intensity-correlation"></a>

### [Intensity Correlation (CORL / CORM)](5_5_measurement_modules.md#intensity-correlation)

| Method | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `enable_linear_correlator` | `enable: bool` | `None` | Activates the strictly linear (CORL) correlation engine. |
| `enable_multi_tau_correlator` | `enable: bool` | `None` | Activates the quasi-logarithmic (CORM) correlation engine. |
| `set_intensity_correlation_channel_a` | `channel: int` | `None` | Selects the first input; use `_channel_b(channel)` for the second. Equal channel IDs request autocorrelation. |
| `set_intensity_correlation_bin_width` | `bin_width_in_8ns_units: int` | `None` | Public unit remains 8 ns. For example, 125 selects 1000 ns; native translates for the active hardware profile. |
| `set_intensity_correlation_num_bins` | `num_bins: int` | `None` | Selects integration sample depth, not the 80 returned lag points. |

Choose the result span and run end with `set_result_span(NEXATOM_RESULT_PROCESSOR_CORRELATION, ...)` and `set_run_end(...)` before enabling and starting the correlators; the setting covers CORL and CORM together. Where g² is undefined the values are `NaN`. `normalization_valid` qualifies the returned g² values; a nonempty array is not sufficient.

<a id="dynamic-light-scattering-analysis"></a>

### [Dynamic Light Scattering (DLS) analysis](5_5_measurement_modules.md#dynamic-light-scattering-analysis)

Extracts hydrodynamic nanoparticle sizes from the CORM output via Cumulant Analysis.

| Method | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `enable_dls_analysis` | `enable: bool` | `None` | Activates the DLS fitting algorithms on the correlation data. |
| `set_dls_experimental_conditions` | `wavelength_nm: float`, `angle_deg: float`, `temperature_c: float`, `viscosity_mPa_s: float`, `refractive_index: float` | `None` | Supplies optical and solvent parameters; temperature input is Celsius. |
| `set_dls_fit_range` | `start_index: int`, `end_index: int` | `None` | Selects a region by correlation-array indices, not by lag times. |
| `enable_dls_cumulant_analysis` | `enable: bool` | `None` | Enables cumulant analysis. |
| `set_dls_fitting_control` | `tolerance: float`, `max_iterations: int`, `initial_beta: float`, `initial_baseline: float` | `None` | Sets solver limits and initial estimates. |

<a id="fluorescence-correlation-spectroscopy"></a>

### [Fluorescence Correlation Spectroscopy (FCS)](5_5_measurement_modules.md#fluorescence-correlation-spectroscopy)

Decomposes confocal molecular diffusion kinetics and concentrations.

| Method | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `enable_fcs_analysis` | `enable: bool` | `None` | Activates the FCS decomposition solver. |
| `set_fcs_confocal_volume` | `omega_xy_um: float`, `omega_z_um: float` | `None` | Defines the lateral/axial confocal dimensions in micrometres. |
| `set_fcs_experimental_conditions` | `temperature_celsius: float`, `viscosity_mPa_s: float`, `wavelength_nm: float`, `calibration_diffusion_um2_s: float` | `None` | Supplies solvent, excitation and diffusion-calibration parameters. |
| `set_fcs_fit_range` | `start_index: int`, `end_index: int` | `None` | Restricts analysis by lag-array indices. |
| `set_fcs_fitting_control` | `tolerance: float`, `max_iterations: int`, `initial_n: float`, `initial_tau_d: float` | `None` | Sets convergence and initial estimates. |

<a id="diffuse-correlation-spectroscopy"></a>

### [Diffuse Correlation Spectroscopy (DCS)](5_5_measurement_modules.md#diffuse-correlation-spectroscopy)

Models deep-tissue hemodynamics using the semi-infinite photon diffusion equation.

| Method | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `enable_dcs_analysis` | `enable: bool` | `None` | Activates DCS in-vivo blood flow modeling. |
| `set_dcs_tissue_properties` | `mu_a: float`, `mu_s_prime: float`, `source_detector_separation_cm: float` | `None` | Supplies absorption/reduced scattering coefficients (per cm) and source-detector separation (cm). |
| `set_dcs_model_parameters` | `anisotropy_g: float`, `tissue_n: float`, `wavelength_nm: float` | `None` | Supplies anisotropy, refractive index and wavelength. |
| `set_dcs_fit_range` | `start_index: int`, `end_index: int` | `None` | Restricts analysis by lag-array indices. |

DLS/FCS/DCS results are embedded in the CORM callback's `analysis_result`. Only one technique is selected for that exported result; see [fitting/result validity](../06_in_depth_guides/6_4_curve_fitting_and_analysis_pipelines.md). The main fit record supplies validity and goodness-of-fit; nested fields are not independently proof of convergence.

### DTC output controls

For a profile authorizing DTC, call `apply_dtc_output(configuration, timeout_ms=1000)`, inspect the returned hardware apply result, then use `set_dtc_global_enable(enable)` as intended. `get_dtc_status()` inspects current state; `clear_dtc_status()` clears the supported status condition. A rejected apply raises `NexatomDtcApplyRejected` with the diagnostic result. Do not treat a proposed configuration as applied.
