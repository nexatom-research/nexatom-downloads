# 2.10 Counts per second (CPS)

## CPS integration window selection

`set_cps_period_selector(0)` selects a 1000 ms measurement period; selector 1 selects 100 ms where supported. The corresponding getter reports the configured selector. Enable the intended system/output path and register the CPS callback before observing results.

## CPS data structure

`NexatomCpsData` / `nexatom_cps_data_t` contains `total_count`, `measurement_period_ms`, `num_channels` and `counts`.

**The historically named `counts` values are already rates in Hz. Do not multiply them again by `1000 / measurement_period_ms`.** Native performs that scaling. Hidden public channels are zeroed and totals are restricted to enabled public-channel data.

Use the reported valid channel extent and native channel authority. The integration period describes the measurement, not callback arrival spacing or proof that USB transport dropped no data.

```python
# Called by a registered CPS callback; the public Python binding owns this record.
def on_cps(data):
    rates_hz = [int(x) for x in data.counts[:int(data.num_channels)]]
    print(rates_hz)  # Already Hz, including when the period is 100 ms.
```

[Device operation](index.md) · [Realtime tutorial](../03_tutorials/3_3_realtime_cps_and_telemetry_streaming.md)
