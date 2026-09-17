# 2.6 Test signal

## Enabling the internal test pulse generator

Use `enable_channel_test_pulse(channel, enable)` on authorized inputs. The acquisition templates configure pulses while quiet and explicitly disable them during cleanup. Their `--internal-test` option records that the source was internal.

## Configuring test pulse parameters

`set_channel_test_pulse_params(channel, period, width)` takes **clock cycles**, not a frequency or picoseconds. Read these profile fields:

- `test_pulse_clock_hz`
- `test_pulse_min_period_cycles`
- `test_pulse_max_period_cycles`

For a supported clock, frequency is clock/period and pulse duration is width/clock. Keep period within the reported range, with `0 < width < period`. A missing/zero clock or unusable limits means no usable pulse plan; do not substitute a hard-coded 125 MHz clock.

The shared setup in the packaged templates derives a plan near 100 kHz. Internal pulses test acquisition, file persistence and controlled digital timing relationships. They bypass the external analog-input test and do not establish analog calibration or maximum throughput.

[Device operation](index.md) · [Quick start](../01_getting_started/1_2_quick_start.md)
