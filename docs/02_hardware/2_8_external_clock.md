## External Clock Input

A supporting device/image can use a fixed 10 MHz external reference to share a frequency reference with laboratory equipment. Check `device.get_capabilities().supports_external_clock` after native runtime entry. A connector, telemetry bit or defined API method does not by itself establish support. Sharing a reference also does not automatically align timestamp epochs between instruments.

External clock control is available in both C and Python. The native API rejects an unsupported request. Follow the supplied instrument's electrical specifications for the reference input; the frequency contract does not specify allowable connector voltages.

| Rule | UTT810 | UTT160810 |
|---|---|---|
| Support | Needs `NEXATOM_TT_FEATURE_FIXED_10MHZ_REFERENCE` in the profile. `Z080_005` announces it; `Z080_004` does not | Needs the same feature bit. Current UTT160810 images, `K168_005` included, do not announce it |
| Without the feature | `request_sync_clock()` and `set_reference_auto_relock()` return `NEXATOM_ERROR_NOT_SUPPORTED` and write nothing | Same |
| Acquisition running | Allowed. Expect a few ms without data; raw time tags after the switch start a new time-base segment | Returns `NEXATOM_ERROR_ACQUISITION_RUNNING` while the system is enabled; call `enable_system(False)` first |
| File save running | A change returns `NEXATOM_ERROR_DEVICE_BUSY` and writes nothing (the switch would leave a gap in the file); repeating the request already in force returns `NEXATOM_SUCCESS` | Same |
| REF IN lost | The board falls back to its own clock by itself | The board falls back to its on-board oscillator by itself |

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
device.enable_system(False)   # Required on the UTT160810; optional on the UTT810.
device.request_sync_clock(True)
# Wait for valid telemetry to report the requested external reference active.
```

<a id="reference-clock-status"></a>

### Reference clock status

`get_reference_clock_status()` (C: `nexatom_tt_get_reference_clock_status()`, 24-byte record) answers "does the board run on REF IN?" from the latest telemetry. It is cache-only and never sends a request. Until the first telemetry of the session it returns `NEXATOM_ERROR_TIMEOUT`. During raw time-tag output no telemetry arrives, so the status stays the one from before raw output.

| `state` | Value | Meaning |
|---|---|---|
| `NEXATOM_TT_REFERENCE_CLOCK_STATE_OFF` | 0 | The external reference is not requested; the board runs on its own clock |
| `_WAITING` | 1 | Requested; the board still runs on its own clock: REF IN is absent, or the switch is in progress (a few ms) |
| `_LOCKED` | 2 | Requested; the board runs on REF IN with 10 MHz present |
| `_LOST` | 3 | Requested; the board ran on REF IN and lost it, and now runs on its own clock |
| `_LOCK_FAILED` | 4 | Requested, but the board gave up locking; `fault_code` says why. Release the request and request it again to retry |

`fault_code` (valid when `flags` has `NEXATOM_TT_REFERENCE_CLOCK_FAULT_CODE_VALID`) is `NEXATOM_TT_REFERENCE_FAULT_NONE` (0), `_PROFILE` (1), `_INPUT` (2), `_SI_LOCK` (3), `_FREQUENCY` (4) or `_CLOCK` (5). `flags` also carries the telemetry bits `SYNC_REQUESTED`, `SYNC_ACTIVE`, `SYNC_LOCKED`, `SYNC_LOST_LATCHED` and `SYNC_LOCK_FAILED_LATCHED`. A board that locked again after a loss reads `LOCKED` with the lost latch still set; each latch clears when the request is released. `telemetry_sequence` names the telemetry packet used: to wait for a switch, poll until the sequence moves on and the state settles.

A recording that is running when the board loses REF IN ends with the end reason `REFERENCE_LOST` (see [File saving and data export](../06_in_depth_guides/6_1_file_saving_and_data_export.md)).

<a id="automatic-relock"></a>

### Automatic relock

`set_reference_auto_relock(enabled)` (C: `nexatom_tt_set_reference_auto_relock()`) chooses what the board does when REF IN comes back after a loss:

- `True` (the default): the board locks to REF IN again by itself.
- `False`: the board stays on its own clock until the application releases the request and requests it again (`request_sync_clock(False)`, then `request_sync_clock(True)`).

The decision is made on the board, so it works without the host. Neither acquisition nor a file save refuses this call. It needs the same feature bit as `request_sync_clock()`. Disconnect switches automatic relock back on.

<a id="reference-clock-output"></a>

### Reference clock output (REF OUT)

`enable_reference_clock_output(enable)` (C: `nexatom_tt_enable_reference_clock_output()`) drives a 10 MHz clock locked to the board's time base on REF OUT, so that other instruments can lock to this one. It is the opposite direction to `request_sync_clock()`, and the two are independent.

- It needs `NEXATOM_TT_FEATURE_REFERENCE_CLOCK_OUTPUT` in the profile on either board; otherwise it returns `NEXATOM_ERROR_NOT_SUPPORTED` and writes nothing. UTT810 images without this bit keep REF OUT always on, and the host cannot switch it.
- Nothing needs to be stopped first; switching it does not disturb a capture.
- **UTT810:** REF OUT is a dedicated connector.
- **UTT160810:** REF OUT is SMA output 8, which it shares with pulse output 7. While the clock is on, pulse output 7 still runs inside the board but does not reach the connector, so scheduling it returns `NEXATOM_ERROR_NOT_SUPPORTED`. Outputs 0–6 are unaffected. Switching the clock off returns the connector to output 7.
- Disconnect switches the clock output off.

```python
device.enable_reference_clock_output(True)    # 10 MHz on REF OUT
```

### Sync-clock status monitoring via telemetry

Because the clock transition is asynchronous and managed autonomously by the FPGA, the definitive state of the clock path must be monitored via the device's realtime telemetry stream.

The `nexatom_telemetry_data_t` structure provides boolean fields that track the internal clock state machine:

| Field | Description |
|---|---|
| `sync_clock_requested` | `true` if the host has issued a request for the external clock path via `nexatom_tt_request_sync_clock()`. |
| `sync_clock_locked` | `true` if the hardware PLL has successfully locked onto a valid external clock signal. |
| `sync_clock_active` | `true` if the hardware has successfully transitioned and is actively operating on the external clock domain. |
| `sync_clock_lost_latched` | `true` once the board lost REF IN while running on it; cleared when the request is released. |
| `sync_clock_lock_failed_latched` | `true` once the board gave up locking to REF IN; cleared when the request is released. |

**Verification.** Wait for valid, current telemetry reporting the requested reference active and locked before commencing acquisition. For the versioned telemetry view, check `HAS_COMMON_STATUS` before inspecting the `SYNC_REQUESTED`, `SYNC_ACTIVE`, `SYNC_LOCKED`, `SYNC_LOST_LATCHED` and `SYNC_LOCK_FAILED_LATCHED` bits in `sync_status_flags`; the view's `reference_fault_code` gives the fault code. `get_reference_clock_status()` above combines these into one state. A field absent from the received layout is unavailable, not a negative lock result. Use a bounded wait and report failure if the reference never becomes usable; do not wait indefinitely or treat an old cached frame as the new request's response.

This verifies the hardware's reported clock state. Applications that require absolute phase or timestamp alignment need a corresponding synchronization experiment; reference-clock lock alone does not establish that stronger claim. See [Telemetry and Diagnostics](2_11_telemetry.md) for the cache/request distinction.
