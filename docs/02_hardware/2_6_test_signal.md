## Test Signal

The UTT810 features an internal test pulse generator capable of injecting synthetic signals into the time-tagging pipeline independently for each channel. This is designed for validating the software data path, testing coincidence logic without optical sources, and verifying SDK integration.

### [Enabling the internal test pulse generator](2_6_test_signal.md#enabling-the-internal-test-pulse-generator)

When the test pulse is enabled for a given channel, the hardware ignores the physical analog input on the SMA connector and injects the internally generated digital signal directly into the FPGA discriminator logic. The channel must be configured with a valid threshold and edge type for the synthetic pulses to generate time tags.

#### C API

```c
nexatom_tt_enable_channel_test_pulse(
    nexatom_tt_handle device,
    uint8_t channel,
    bool enable
);
```

#### Python

```python
device.enable_channel_test_pulse(channel=0, enable=True)
```

### [Configuring test pulse parameters](2_6_test_signal.md#configuring-test-pulse-parameters)

The frequency and duty cycle of the test pulse are fully configurable. Parameters are specified in units of the FPGA's 125 MHz internal clock cycles. At 125 MHz, one clock cycle is exactly 8 nanoseconds.

| Parameter | Valid Range (Cycles) | Physical Equivalent |
|---|---|---|
| `period_cycles` | `1` to `125000000` | Period of the signal (8 ns to 1.0 s) |
| `width_cycles` | `1` to `125000000` | Pulse high-time duration (8 ns to 1.0 s) |

> **Constraint.** The `width_cycles` must be strictly less than `period_cycles` to generate a valid toggling signal.

#### C API

```c
nexatom_tt_set_channel_test_pulse_params(
    nexatom_tt_handle device,
    uint8_t channel,
    uint32_t period_cycles,
    uint32_t width_cycles
);
```

#### Python

```python
# Configure a 1 MHz signal (1000 ns period) with 50% duty cycle (500 ns width)
# 1000 ns / 8 ns = 125 cycles
#  500 ns / 8 ns =  62 cycles (rounded)

device.set_channel_test_pulse_params(
    channel=0, 
    period_cycles=125, 
    width_cycles=62
)
```
