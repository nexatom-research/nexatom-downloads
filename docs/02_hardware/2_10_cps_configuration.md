## CPS (Count Rate) Configuration

The UTT810 hardware continuously calculates the Counts Per Second (CPS) for all input channels. This data is emitted periodically and delivered to the host when the device is operating in `REALTIME_DATA` output mode.

### [CPS integration window selection](2_10_cps_configuration.md#cps-integration-window-selection)

The integration window (measurement period) over which the hardware accumulates events is configurable. Shorter windows provide faster update rates at the cost of higher variance in low-count scenarios.

| Selector Value | Integration Window | Update Rate |
|---|---|---|
| `0` | 1000 ms (default) | 1 Hz |
| `1` | 100 ms | 10 Hz |

#### C API

```c
/* Set the CPS period to 100 ms */
nexatom_tt_set_cps_period_selector(device, 1);

/* Query the current selector value */
uint32_t current_selector;
nexatom_tt_get_cps_period_selector(device, &current_selector);
```

#### Python

```python
# Set the CPS period to 100 ms
device.set_cps_period_selector(selector=1)
```

*(Note: There is currently no `get_cps_period_selector` getter wrapped in the Python class).*

### [CPS data structure](2_10_cps_configuration.md#cps-data-structure)

When a registered `nexatom_count_rate_callback` is invoked, it receives the `nexatom_cps_data_t` structure. In Python, this is mapped directly to the `NexatomCpsData` `ctypes` structure.

#### `nexatom_cps_data_t`

| Field | Type | Description |
|---|---|---|
| `total_count` | `uint32_t` | The sum of all time-tag events recorded across all channels during the integration window. |
| `measurement_period_ms` | `uint32_t` | The actual measurement window in milliseconds (1000 or 100). |
| `num_channels` | `uint8_t` | Number of active channels supported by the device model (8 for the UTT810). |
| `counts` | `uint32_t[8]` | Array of per-channel event counts. Only indices `0` through `num_channels - 1` contain valid data. |

> **Throughput note.** The counts reported in the CPS data structure reflect the exact number of events detected by the hardware's discriminator logic, regardless of whether the USB connection or host PC experienced bandwidth bottlenecks or data drops during the period.
