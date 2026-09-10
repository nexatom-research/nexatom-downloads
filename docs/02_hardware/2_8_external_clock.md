## External Clock Input

The UTT810 supports synchronization to an external reference clock, allowing multiple devices or disparate laboratory systems to share a common, unified time base.

> **Python Wrapper Support.** External clock configuration is currently only exposed in the native C API. It is not yet wrapped in the `NexatomDevice` Python class.

### [Requesting external sync-clock operation](2_8_external_clock.md#requesting-external-sync-clock-operation)

Switching the hardware to an external clock is an asynchronous operation. The host software issues a request, and the device's internal phase-locked loop (PLL) attempts to acquire and lock onto the external signal before dynamically switching clock domains.

#### C API

```c
nexatom_tt_request_sync_clock(
    nexatom_tt_handle device,
    bool enable
);
```

Calling this function with `enable = true` does not guarantee immediate synchronization. The system will only switch to the external clock if a valid signal is detected and locked. If `enable = false`, the hardware immediately requests a return to the internal oscillator.

### [Sync-clock status monitoring via telemetry](2_8_external_clock.md#sync-clock-status-monitoring-via-telemetry)

Because the clock transition is asynchronous and managed autonomously by the FPGA, the definitive state of the clock path must be monitored via the device's realtime telemetry stream.

The `nexatom_telemetry_data_t` structure provides three boolean fields that track the internal clock state machine:

| Field | Description |
|---|---|
| `sync_clock_requested` | `true` if the host has issued a request for the external clock path via `nexatom_tt_request_sync_clock()`. |
| `sync_clock_locked` | `true` if the hardware PLL has successfully locked onto a valid external clock signal. |
| `sync_clock_active` | `true` if the hardware has successfully transitioned and is actively operating on the external clock domain. |

**Verification.** The `sync_clock_active` flag is the sole definitive indicator that subsequent time tags and data packets are synchronized to the external reference. Host software should poll telemetry and wait for `sync_clock_active == true` before commencing data acquisition.
