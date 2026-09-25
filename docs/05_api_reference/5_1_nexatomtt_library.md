## The NexatomTT Library

This section defines the fundamental constants, enumerations, and limit bounds utilized throughout the SDK. These values dictate hardware capabilities, buffer sizings, and operational state machines.

### [Constants and limits](5_1_nexatomtt_library.md#constants-and-limits)

| Constant | Value | Description |
|----------|-------|-------------|
| `NEXATOM_MAX_CHANNELS` | 8 | Capacity of legacy C/Python channel arrays, not a universal physical-device count. |
| `NEXATOM_MAX_STRING_LENGTH` | 256 | Maximum byte size for string buffers (e.g., serial numbers, error messages). |
| `NEXATOM_MAX_HISTOGRAM_BINS` | 1024 | The absolute maximum size of the TIHI hardware histogram array. |
| `NEXATOM_MAX_CONFIGURABLE_TIME_HISTOGRAM_BINS` | 1024 | Binding ceiling. Use `get_capabilities().max_histogram_bins` for the active device; a profile may permit only 1023. |
| Correlation array capacity | 80 | CORL/CORM returned lag points; do not confuse this with integration sample depth. |
| `NEXATOM_CONFIG_DUMP_MAX_REGISTERS` | 128 | Maximum number of register address-value pairs returned in a config dump. |
| `NEXATOM_MAX_CHANNEL_INPUT_DELAY_PS` | 256000 | Binding ceiling (256 ns); the active profile's `max_channel_input_delay_ps` remains authoritative. |

Use the profile's `effective_public_tdc_mask`, `feature_flags` and `supported_output_mode_mask` to select channels and operations. A constant reserves storage or identifies a feature; it does not enable that feature on every instrument. `version()` reports the native library version, not the SDK archive's preview release label.

<a id="error-codes"></a>

### [Error codes (`nexatom_error_code_t`)](5_1_nexatomtt_library.md#error-codes)

C operations returning `nexatom_error_code_t` use a signed 32-bit integer indicating success or the specific mode of failure. The Python wrapper checks these operations and raises `NexatomError` on failure. Preserve `last_error_message()` promptly; another native operation can replace it. Functions with other return types follow their own documented convention.

*   `0`: `NEXATOM_SUCCESS`
*   `-1`: `NEXATOM_ERROR_INVALID_PARAMETER`
*   `-2`: `NEXATOM_ERROR_NOT_CONNECTED`
*   `-3`: `NEXATOM_ERROR_ALREADY_CONNECTED`
*   `-4`: `NEXATOM_ERROR_CONNECTION_FAILED`
*   `-5`: `NEXATOM_ERROR_TIMEOUT`
*   `-6`: `NEXATOM_ERROR_DEVICE_BUSY`
*   `-7`: `NEXATOM_ERROR_ACQUISITION_RUNNING`
*   `-8`: `NEXATOM_ERROR_NO_ACQUISITION`
*   `-9`: `NEXATOM_ERROR_CONFIGURATION_FAILED`
*   `-10`: `NEXATOM_ERROR_MEMORY_ALLOCATION`
*   `-11`: `NEXATOM_ERROR_FILE_IO`
*   `-12`: `NEXATOM_ERROR_NOT_SUPPORTED`
*   `-13`: `NEXATOM_ERROR_CALIBRATION_FAILED`
*   `-14`: `NEXATOM_ERROR_MODE_RESTORE_FAILED`
*   `-99`: `NEXATOM_ERROR_INTERNAL` (Fatal SDK state violation)

<a id="device-states"></a>

### [Device states (`nexatom_tt_state_t`)](5_1_nexatomtt_library.md#device-states)

The internal state machine governs which commands are valid at any given time.
*   `DISCONNECTED` (0)
*   `CONNECTING` (1)
*   `CONNECTED` (2): Connected session; use `connect_runtime()` and profile authorization to establish measurement readiness.
*   `CONFIGURING` (3): Actively writing to FPGA registers.
*   `ACQUIRING` (4): Acquisition state. System enable, output selection and starting an individual engine remain distinct operations.
*   `STOPPING` (5)
*   `ERROR` (6)

<a id="result-model"></a>

### [Result model (`nexatom_result_span_t`, `nexatom_result_status_t`, `nexatom_run_end_kind_t`)](5_1_nexatomtt_library.md#result-model)

TIHI, MFCO, the correlators (CORL and CORM) and Fast TIHI share one result model. The hardware measures in batches that the SDK programs (1 s for TIHI, MFCO and Fast TIHI; `num_bins × T` for the correlators); the host sums whole batches into the published results. The aggregation modes (`ACCUMULATE`, `REPLACE`, `AVERAGE`), `nexatom_aggregation_mode_t` and the `set_*_aggregation_mode()` and `set_*_stop_conditions()` calls no longer exist.

Each call takes a processor: `NEXATOM_RESULT_PROCESSOR_TIME_HISTOGRAM` (0), `NEXATOM_RESULT_PROCESSOR_MULTIFOLD_COINCIDENCE` (1), `NEXATOM_RESULT_PROCESSOR_CORRELATION` (2, CORL and CORM) or `NEXATOM_RESULT_PROCESSOR_FAST_TIME_HISTOGRAM` (3). A processor that the running image lacks returns `NEXATOM_ERROR_NOT_SUPPORTED`.

| Python method | What it sets |
| --- | --- |
| `set_result_span(processor, span, block_ms=0)` | What a result covers. `NEXATOM_RESULT_SPAN_WHOLE_RUN` (0, default): the total since Start, `clear_result()` or a restarting change, republished about once a second. `NEXATOM_RESULT_SPAN_BLOCK` (1): tumbling blocks, each published once at the first batch edge at or after `block_ms`; the next block starts empty. |
| `set_run_end(processor, kind, value=0)` | When the run ends by itself. `NEXATOM_RUN_END_NONE` (0, default): run until Stop. `NEXATOM_RUN_END_TIME` (1): `value` ms of measurement time. `NEXATOM_RUN_END_COUNT` (2): `value` detected events. |
| `clear_result(processor)` | Discards what the host has summed; the next result, block and run-end count start from this call. The hardware keeps running. |

Changing the span or the run end restarts the result, as does changing the TIHI channels, bidirectional or first-stop mode. Writing the current value does not.

**Moving from the old modes.** `ACCUMULATE` is now `NEXATOM_RESULT_SPAN_WHOLE_RUN` (the default): each result is the running total. `REPLACE` is now `NEXATOM_RESULT_SPAN_BLOCK` with `block_ms` equal to the batch length (1000 ms for TIHI and MFCO), so each result holds one batch and the next starts empty. `AVERAGE` has no mode; compute it as below.

**Mean per batch and rates (replacing `AVERAGE`).** Results are sums; there is no averaging mode. For the mean counts per hardware batch, divide a `WHOLE_RUN` result's counts by its `packets_accumulated` (TIHI, MFCO; `windows_accumulated` for Fast TIHI). For a rate, divide the counts by `live_time_ms / 1000`, the measurement time the result contains. Prefer the rate when `live_time_exact` is 0: a batch was ended by Stop and is shorter than the others, so a per-batch mean would be biased low. Correlator results (CORL, CORM) are normalised g² curves pooled over their batches, not counts; do not divide them.

Every result carries `result_status`:

*   `RUNNING` (0): `WHOLE_RUN` progress update; the run continues.
*   `BLOCK_COMPLETE` (1): a `BLOCK` result reached its length; the next block has started.
*   `RUN_COMPLETE` (2): the run-end condition was met and the SDK stopped the processor. Last result of the run.
*   `STOPPED` (3): final partial result after Stop. Last result of the run.

After Stop, exactly one `STOPPED` result arrives per processor, within 2 s. A Fast TIHI `STOPPED` result can carry a newer `run_id` than the last one seen; compare `run_id` wrap-safely. See [result model fields](5_3_data_structures.md#result-model-fields).

<a id="acquisition-done-status"></a>

### [Acquisition done status (`nexatom_acquisition_done_status_t`)](5_1_nexatomtt_library.md#acquisition-done-status)

Included in TIHI/MFCO terminal results and the applicable correlation status. These are protocol reason codes, not a sequential success/failure enum. Use the exported symbols:

| Constant | Value | Interpretation |
| --- | --- | --- |
| `NEXATOM_ACQ_REASON_CODE_0X0` / `NEXATOM_ACQ_NORMAL_COMPLETION` | `0x0` | Retained reason-code/compatibility name; interpret with the measurement contract. |
| `NEXATOM_ACQ_HARDWARE_OVERFLOW` | `0x1` | Hardware overflow. |
| `NEXATOM_ACQ_MANUAL_STOP` | `0x2` | User-issued stop. |
| `NEXATOM_ACQ_STOP_COUNT_REACHED` / `NEXATOM_ACQ_EVENT_COUNT_REACHED` | `0x4` | Count limit reached. |
| `NEXATOM_ACQ_TIME_DURATION_REACHED` | `0x5` | Duration limit reached. |
| `NEXATOM_ACQ_REASON_CODE_0X6` / `NEXATOM_ACQ_DIVIDEND_OVERFLOW` | `0x6` | Retained reason-code/compatibility name; do not silently label as success. |
| `NEXATOM_ACQ_UNKNOWN` | `0xff` | Unknown completion reason. |

Also retain MFCO versioned quality flags and correlation normalization validity. A terminal callback can report an overflow or invalid normalization; receiving it does not establish a scientifically usable measurement.
