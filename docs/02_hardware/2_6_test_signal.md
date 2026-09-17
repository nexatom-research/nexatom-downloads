## Test Signal

The UTT810 features an internal test pulse generator capable of injecting synthetic signals into the time-tagging pipeline independently for each channel. This is designed for validating the software data path, testing coincidence logic without optical sources, and verifying SDK integration.

### Enabling the internal test pulse generator

Enabling the test pulse selects an internally generated digital signal for that channel's time-tagging path. It is useful without an external detector, but it does not validate the analog input, discriminator threshold or hysteresis. Configure channel settings explicitly so the later external-input experiment is reproducible; keep the test source disabled until callbacks or file savers are ready.

#### C API

```c
nexatom_error_code_t nexatom_tt_enable_channel_test_pulse(
    nexatom_tt_handle device,
    uint8_t channel,
    bool enable
);
```

#### Python

```python
device.enable_channel_test_pulse(channel=0, enable=True)
```

### Configuring test pulse parameters

The period and high-time are configured in clock cycles. Read `test_pulse_clock_hz`, `test_pulse_min_period_cycles` and `test_pulse_max_period_cycles` from the native device profile; a clock frequency must not be inferred from the model name or copied from another image. With a 125 MHz reported clock, a cycle is 8 ns; with 250 MHz it is 4 ns.

| Parameter | Valid Range (Cycles) | Physical Equivalent |
|---|---|---|
| `period_cycles` | Profile minimum through profile maximum | `period_cycles / test_pulse_clock_hz` seconds |
| `width_cycles` | At least 1 and less than the period | `width_cycles / test_pulse_clock_hz` seconds |

> **Constraint.** Choose `1 <= width_cycles < period_cycles`. Modern profiles require a period of at least 2 cycles and accept a maximum of 134,217,727 cycles. The profile supplies the applicable bounds; a zero/absent clock is not permission to substitute a guessed value.

#### C API

```c
nexatom_error_code_t nexatom_tt_set_channel_test_pulse_params(
    nexatom_tt_handle device,
    uint8_t channel,
    uint32_t period_cycles,
    uint32_t width_cycles
);
```

#### Python

```python
# Configure approximately 100 kHz with a half-period high time.
profile = device.get_device_profile()
clock_hz = int(profile.test_pulse_clock_hz)
if clock_hz <= 0:
    raise RuntimeError("No test-pulse timing is available for this device")
period = round(clock_hz / 100_000)
minimum = max(2, int(profile.test_pulse_min_period_cycles))
maximum = int(profile.test_pulse_max_period_cycles)
if not minimum <= period <= maximum:
    raise ValueError("Requested test frequency is outside the profile limits")
width = period // 2

device.enable_channel_test_pulse(0, False)
device.set_channel_test_pulse_params(0, period, width)
# Open the saver/register callbacks before selecting the source and starting.
print(f"Actual frequency: {clock_hz / period:g} Hz")
```

At 125 MHz, a 100 kHz setting uses 1,250 cycles and adjacent pulses on one channel are 10 microseconds apart. Equal periods on two channels do not establish their relative phase: do not assume coincident edges merely because both generators use the same numbers.

For a TIHI measurement of adjacent pulses, choose a histogram span (`bin_width_ps * num_bins`) large enough to include that separation. For MFCO choose the coincidence window for the event grouping you want to observe. A nonzero CPS rate can coexist with an empty histogram when its window misses the expected delays.

For example, 256 TIHI bins of 64,000 ps span 16.384 microseconds, covering the 10 microsecond interval. A 12,000,000 ps MFCO window spans 12 microseconds. These settings illustrate a digital-path check; a two-channel peak or fold still depends on channel selection and phase. Always disable every enabled test source during cleanup before using external signals again.
