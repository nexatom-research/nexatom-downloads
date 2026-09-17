# 5.5 Measurement modules

Prepare supported controls while quiet, then explicitly enable/start the intended engines. Profile validation and native errors remain authoritative for every method below. Retain status/quality metadata when interpreting a callback.

## Time histogram

`set_time_histogram_channels(start_channel, stop_channel)`, `set_time_histogram_bin_width(bin_width_ps)` and `set_time_histogram_num_bins(num_bins)` configure normal TIHI. Set stop count/duration, aggregation, first-stop/bidirectional behaviour, and callback before `enable_time_histogram(True)` / `start_time_histogram()`. Background and fitting controls include `set_tihi_background_method`, `set_tihi_signal_region`, `set_tihi_background_region`, `enable_tihi_fitting`, fitting parameters and model. See [TIHI C reference](../07_c_api/7_9_tihi.md).

Fast TIHI is a separate capability: `start_fast_tihi(configuration)` accepts the versioned V1/V2 configuration, `stop_fast_tihi()` stops it, and its callback carries the versioned histogram. Do not substitute it for normal TIHI based only on a model name.

## Multifold coincidence

`set_multifold_coincidence_channels(channels)` accepts an iterable of channel IDs for up to eight MFCO slots. `0xff` disables a slot; omitted slots are padded with `0xff`. It does not accept a single channel bitmask or an array of enable booleans. Current returned pattern bins are analyzed in software; slot/mask metadata is not a guarantee of physical input gating.

Set the window in ps, stop conditions, aggregation, optional pattern filter and background method, then enable/start MFCO. `set_multifold_coincidence_pattern_filter(requirements)` takes exactly eight requirements. Use documented requirement constants. See [pattern helpers](5_6_mfco_pattern_analysis_helpers.md) and [MFCO reference](../07_c_api/7_10_mfco.md).

## Intensity correlation

Use separate `set_intensity_correlation_channel_a(channel)` and `_channel_b(channel)` methods. `set_intensity_correlation_bin_width(bin_width_in_8ns_units)` retains the public 8 ns unit; native translates for the active profile. `set_intensity_correlation_num_bins(num_bins)` configures integration sample depth, not the 80 returned lag points. Configure stop conditions and linear/multi-tau aggregation, enable the desired correlators, then start. Check normalization validity. See [correlation](../07_c_api/7_11_correlation.md).

## Scientific analysis

| Analysis | Key Python signatures/units |
| --- | --- |
| DLS | `set_dls_fit_range(start_index, end_index)`; `set_dls_experimental_conditions(wavelength_nm, angle_deg, temperature_c, viscosity_mPa_s, refractive_index)`; bounds, cumulant enable and fitting control |
| FCS | `set_fcs_fit_range(start_index, end_index)`; `set_fcs_confocal_volume(omega_xy_um, omega_z_um)`; `set_fcs_experimental_conditions(temperature_celsius, viscosity_mPa_s, wavelength_nm, calibration_diffusion_um2_s)` |
| DCS | `set_dcs_fit_range(start_index, end_index)`; `set_dcs_tissue_properties(mu_a, mu_s_prime, source_detector_separation_cm)`; `set_dcs_model_parameters(anisotropy_g, tissue_n, wavelength_nm)` |

Each family has `enable_*_analysis` and fitting controls. These are host analysis models, not a guarantee of scientifically valid results from arbitrary signals. See [fitting/result validity](../06_in_depth_guides/6_4_curve_fitting_and_analysis_pipelines.md).

## DTC

For a profile authorizing DTC, use `apply_dtc_output(configuration, timeout_ms=1000)`, inspect the returned apply result, then use `set_dtc_global_enable(enable)` as intended. `get_dtc_status` and `clear_dtc_status` have separate purposes. A rejected apply raises `NexatomDtcApplyRejected` with the returned diagnostic result; do not treat a proposed configuration as applied.

[Python reference](index.md)
