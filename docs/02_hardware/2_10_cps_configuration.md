## CPS (Count Rate) Configuration

The UTT810 hardware accumulates input events over a measurement window. In `REALTIME_DATA` mode the native SDK delivers normalized Counts Per Second (CPS) values to the application. Check that the connected profile advertises CPS support and interpret only the public channel indices available to that profile.

### CPS integration window selection

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

```python
current_selector = device.get_cps_period_selector()
```

The selector is a requested configuration. `measurement_period_ms` in a received packet records the window represented by that packet; use it when interpreting data around a configuration change.

### CPS data structure

When a registered `nexatom_count_rate_callback` is invoked, it receives the `nexatom_cps_data_t` structure. In Python, this is mapped directly to the `NexatomCpsData` `ctypes` structure.

#### `nexatom_cps_data_t`

| Field | Type | Description |
|---|---|---|
| `total_count` | `uint32_t` | Aggregate normalized count rate in events/second, despite the historical field name. |
| `measurement_period_ms` | `uint32_t` | The actual measurement window in milliseconds (1000 or 100). |
| `num_channels` | `uint8_t` | Number of entries in the public CPS record; it is not a grant to configure physical channels. |
| `counts` | `uint32_t[8]` | Per-channel normalized rates in events/second. Use the packet layout and effective public mask when selecting entries. |

For example, one event in a 100 ms hardware window is delivered as `counts[channel] == 10` CPS. Do not divide this public value by 0.1 again. The `measurement_period_ms` field remains 100, even though `counts` and `total_count` have been converted to rates.

```python
def on_cps(data):
    # Copy simple values during the callback; do slower work elsewhere.
    rates = tuple(int(data.counts[ch]) for ch in range(int(data.num_channels)))
    print("CPS:", rates, "window(ms):", int(data.measurement_period_ms))

device.set_count_rate_callback(on_cps)
```

> **Throughput note.** CPS describes hardware counting, not the number of raw tags that reached a saved file. A plausible count rate is useful evidence of an active input path, but it cannot prove loss-free USB transport, callback delivery or file capture. Check those outputs separately.
