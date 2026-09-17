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

<a id="aggregation-modes"></a>

### [Aggregation modes (`nexatom_aggregation_mode_t`)](5_1_nexatomtt_library.md#aggregation-modes)

Governs how the SDK processes sequential data payloads from the FPGA before dispatching the user callback. Applies to TIHI, MFCO, and Correlation engines.
*   `ACCUMULATE` (0): New hardware counts are summed directly into the existing host-side histogram.
*   `REPLACE` (1): The host-side histogram is cleared and overwritten by the latest hardware packet.
*   `AVERAGE` (2): Maintains a running statistical average across sequential hardware packets.

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
