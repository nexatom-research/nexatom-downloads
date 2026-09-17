## External Clock Input

A supporting device/image can use a fixed 10 MHz external reference to share a frequency reference with laboratory equipment. Check `device.get_capabilities().supports_external_clock` after native runtime entry. A connector, telemetry bit or defined API method does not by itself establish support. Sharing a reference also does not automatically align timestamp epochs between instruments.

External clock control is available in both C and Python. The native API rejects an unsupported request and requires acquisition to be disabled before changing the reference selection. Follow the supplied instrument's electrical specifications for the reference input; the frequency contract does not specify allowable connector voltages.

### Requesting external sync-clock operation

Switching the hardware to an external reference is asynchronous. The host issues a request; the hardware must detect/lock to the reference and report whether the external path is active. Do not start a measurement based only on a successful request call.

#### C API

```c
nexatom_error_code_t nexatom_tt_request_sync_clock(
    nexatom_tt_handle device,
    bool enable
);
```

Calling with `enable = true` does not guarantee immediate synchronization. Calling with `enable = false` requests the internal reference again; verify the resulting telemetry before resuming a measurement that depends on the time base.

#### Python

```python
if not device.get_capabilities().supports_external_clock:
    raise RuntimeError("This device/image does not support the external reference")
device.enable_system(False)
device.request_sync_clock(True)
# Wait for valid telemetry to report the requested external reference active.
```

### Sync-clock status monitoring via telemetry

Because the clock transition is asynchronous and managed autonomously by the FPGA, the definitive state of the clock path must be monitored via the device's realtime telemetry stream.

The `nexatom_telemetry_data_t` structure provides three boolean fields that track the internal clock state machine:

| Field | Description |
|---|---|
| `sync_clock_requested` | `true` if the host has issued a request for the external clock path via `nexatom_tt_request_sync_clock()`. |
| `sync_clock_locked` | `true` if the hardware PLL has successfully locked onto a valid external clock signal. |
| `sync_clock_active` | `true` if the hardware has successfully transitioned and is actively operating on the external clock domain. |

**Verification.** Wait for valid, current telemetry reporting the requested reference active and locked before commencing acquisition. For the versioned telemetry view, check `HAS_COMMON_STATUS` before inspecting the `SYNC_REQUESTED`, `SYNC_ACTIVE` and `SYNC_LOCKED` bits in `sync_status_flags`. A field absent from the received layout is unavailable, not a negative lock result. Use a bounded wait and report failure if the reference never becomes usable; do not wait indefinitely or treat an old cached frame as the new request's response.

This verifies the hardware's reported clock state. Applications that require absolute phase or timestamp alignment need a corresponding synchronization experiment; reference-clock lock alone does not establish that stronger claim. See [Telemetry and Diagnostics](2_11_telemetry.md) for the cache/request distinction.
