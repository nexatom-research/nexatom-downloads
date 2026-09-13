## The NexatomTT Library

This section defines the fundamental constants, enumerations, and limit bounds utilized throughout the SDK. These values dictate hardware capabilities, buffer sizings, and operational state machines.

### [Constants and limits](5_1_nexatomtt_library.md#constants-and-limits)

| Constant | Value | Description |
|----------|-------|-------------|
| `NEXATOM_MAX_CHANNELS` | 8 | Maximum physical input channels available on the UTT810. |
| `NEXATOM_MAX_STRING_LENGTH` | 256 | Maximum byte size for string buffers (e.g., serial numbers, error messages). |
| `NEXATOM_MAX_HISTOGRAM_BINS` | 1024 | The absolute maximum size of the TIHI hardware histogram array. |
| `NEXATOM_MAX_CONFIGURABLE_TIME_HISTOGRAM_BINS` | 1023 | The maximum user-settable bin count (reserving 1 bin for overflow). |
| `NEXATOM_MAX_CORRELATION_LAGS` | 80 | The fixed number of correlation lag times computed by the hardware engines. |
| `NEXATOM_CONFIG_DUMP_MAX_REGISTERS` | 128 | Maximum number of register address-value pairs returned in a config dump. |

### [Error codes (`nexatom_error_code_t`)](5_1_nexatomtt_library.md#error-codes)

All C API functions return a signed 32-bit integer indicating success or the specific mode of failure. In the Python API, any negative return value automatically raises a `NexatomError` exception.

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

### [Device states (`nexatom_tt_state_t`)](5_1_nexatomtt_library.md#device-states)

The internal state machine governs which commands are valid at any given time.
*   `DISCONNECTED` (0)
*   `CONNECTING` (1)
*   `CONNECTED` (2): Idle, ready for configuration.
*   `CONFIGURING` (3): Actively writing to FPGA registers.
*   `ACQUIRING` (4): Measurement engines are active (`enable_system(True)`).
*   `STOPPING` (5)
*   `ERROR` (6)

### [Aggregation modes (`nexatom_aggregation_mode_t`)](5_1_nexatomtt_library.md#aggregation-modes)

Governs how the SDK processes sequential data payloads from the FPGA before dispatching the user callback. Applies to TIHI, MFCO, and Correlation engines.
*   `ACCUMULATE` (0): New hardware counts are summed directly into the existing host-side histogram.
*   `REPLACE` (1): The host-side histogram is cleared and overwritten by the latest hardware packet.
*   `AVERAGE` (2): Maintains a running statistical average across sequential hardware packets.

### [Acquisition done status (`nexatom_acquisition_done_status_t`)](5_1_nexatomtt_library.md#acquisition-done-status)

Included in the final callback payload when an active measurement terminates.
*   `NORMAL_COMPLETION` (0): Terminated naturally based on hardware stop conditions.
*   `MANUAL_STOP` (1): Terminated by a user-issued stop command.
*   `EVENT_COUNT_REACHED` (2): Terminated via the `stop_count` limit.
*   `TIME_DURATION_REACHED` (3): Terminated via the `stop_duration_ms` limit.
*   `DIVIDEND_OVERFLOW` (4): Hardware integration registers overflowed.
*   `UNKNOWN` (5)
