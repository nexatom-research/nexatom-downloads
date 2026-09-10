## Synthetic Input Delay

Small, programmable synthetic delays can be applied to individual input channels. This feature is typically used to compensate for differing external cable lengths or to temporally align the arrival of correlated signals prior to multi-channel coincidence processing.

### [Per-channel input delay configuration](2_7_synthetic_input_delay.md#per-channel-input-delay-configuration)

The synthetic input delay is applied dynamically within the FPGA logic immediately following the discriminator.

| Parameter | Range | Resolution |
|---|---|---|
| `delay_ps` | `0` to `4000` | Picoseconds (ps) |

Providing a `delay_ps` value greater than `4000` will result in a `NEXATOM_ERROR_INVALID_PARAMETER` return code in C, or raise a `NexatomError` exception in Python.

#### C API

```c
nexatom_tt_set_channel_input_delay(
    nexatom_tt_handle device,
    uint8_t channel,
    uint32_t delay_ps
);
```

#### Python

```python
# Compensate for ~50 cm extra BNC cable length on channel 1 (~2500 ps delay)
device.set_channel_input_delay(channel=1, delay_ps=2500)
```
