## Measurement Modules

This section documents the advanced mathematical and analytical configuration endpoints. These methods dictate how the internal C++ solvers evaluate data (e.g., background subtraction, curve fitting, and physical modeling) before dispatching the payload to your Python callbacks.

### [Time Interval Histogram (TIHI) advanced config](5_5_measurement_modules.md#time-interval-histogram-config)

| Method | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `set_tihi_background_method` | `method: int` | `None` | Sets the baseline subtraction mode: `NONE`, `USER_CONSTANT`, or `USER_REGION`. |
| `set_tihi_user_background_value` | `value: float` | `None` | Defines the static baseline subtracted from all bins if `USER_CONSTANT` is active. |
| `set_tihi_signal_region` | `start_bin: int`, `end_bin: int` | `None` | Restricts fitting algorithms and SNR calculations to a specific Region of Interest (ROI). |
| `enable_tihi_fitting` | `enable: bool` | `None` | Activates the Levenberg-Marquardt non-linear least squares solver on the active histogram. |
| `set_tihi_fitting_model` | `model: int` | `None` | Selects the target equation: `EXPONENTIAL`, `BI_EXPONENTIAL`, `GAUSSIAN`, `LORENTZIAN`, `STRETCHED`, or `AUTO`. |

### [Multi-Fold Coincidence (MFCO) advanced config](5_5_measurement_modules.md#multi-fold-coincidence-config)

| Method | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `set_multifold_coincidence_pattern_filter` | `channel: int`, `requirement: int` | `None` | Filters the output based on specific channel presence (`DONT_CARE`, `REQUIRED`, or `FORBIDDEN`). |
| `disable_multifold_coincidence_pattern_filter` | *None* | `None` | Resets all channel requirements to `DONT_CARE`. |
| `set_mfco_background_method` | `method: int` | `None` | Sets the noise subtraction mode: `NONE`, `USER_CONSTANT`, or `USER_SELECTED_PATTERN_BIN`. |

### [Intensity Correlation (CORL / CORM)](5_5_measurement_modules.md#intensity-correlation)

| Method | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `enable_linear_correlator` | `enable: bool` | `None` | Activates the strictly linear (CORL) correlation engine. |
| `enable_multi_tau_correlator` | `enable: bool` | `None` | Activates the quasi-logarithmic (CORM) correlation engine. |
| `set_intensity_correlation_channels` | `ch_a: int`, `ch_b: int` | `None` | Assigns the two physical inputs for cross-correlation (or set both to the same channel for auto-correlation). |
| `set_intensity_correlation_bin_width` | `width_8ns: int` | `None` | Sets the base hardware lag resolution (must be a multiple of the 8ns FPGA clock). |

### [Dynamic Light Scattering (DLS) analysis](5_5_measurement_modules.md#dynamic-light-scattering-analysis)

Extracts hydrodynamic nanoparticle sizes from the CORM output via Cumulant Analysis.

| Method | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `enable_dls_analysis` | `enable: bool` | `None` | Activates the DLS fitting algorithms on the correlation data. |
| `set_dls_experimental_conditions` | `wavelength_nm: float`, `angle_deg: float`, `temp_k: float`, `viscosity_cp: float`, `refractive_idx: float` | `None` | Defines the physical parameters of the experimental setup required to extract the diffusion coefficient. |
| `set_dls_fit_range` | `start_lag_ns: int`, `end_lag_ns: int` | `None` | Restricts the Cumulant solver to a specific temporal region of the correlation decay. |

### [Fluorescence Correlation Spectroscopy (FCS)](5_5_measurement_modules.md#fluorescence-correlation-spectroscopy)

Decomposes confocal molecular diffusion kinetics and concentrations.

| Method | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `enable_fcs_analysis` | `enable: bool` | `None` | Activates the FCS decomposition solver. |
| `set_fcs_confocal_volume` | `lateral_radius_nm: float`, `axial_radius_nm: float` | `None` | Defines the optical observation volume ($V_{eff}$) parameters. |
| `set_fcs_experimental_conditions` | `temp_k: float`, `viscosity_cp: float`, `wavelength_nm: float` | `None` | Defines the solvent and excitation constraints. |

### [Diffuse Correlation Spectroscopy (DCS)](5_5_measurement_modules.md#diffuse-correlation-spectroscopy)

Models deep-tissue hemodynamics using the semi-infinite photon diffusion equation.

| Method | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `enable_dcs_analysis` | `enable: bool` | `None` | Activates DCS in-vivo blood flow modeling. |
| `set_dcs_tissue_properties` | `mu_a: float`, `mu_s_prime: float`, `separation_cm: float` | `None` | Defines the absorption coefficient, reduced scattering coefficient, and optode physical distance. |
