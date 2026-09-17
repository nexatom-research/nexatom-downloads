# 2.2 Input channels

## Channel count and indexing

Use zero-based public channel IDs authorized by `effective_public_tdc_mask`. A physical lane count or the legacy eight-element callback array does not authorize additional public inputs. The primary templates validate the entire channel plan before writing it.

## Threshold configuration

`set_channel_threshold(channel, threshold_mv)` uses integer millivolts. Bound the request by native capabilities (the examples also enforce the API ceiling of 2500 mV). Choose values appropriate to the external signal and instrument specification.

## Edge type selection

`set_channel_edge_type(channel, edge_type)` uses 0 for rising and 1 for falling. An edge change may initiate recalibration. Successful command acceptance does not prove recalibration completion or the measured response of an analog input.

## Input hysteresis

`set_channel_hysteresis(channel, hysteresis_mv)` accepts the native API's 0–175 mV range where supported. This is a control range, not a complete specification of input voltage tolerance or comparator behaviour.

## Channel routing

`set_channel_routing(source_channel, target_channel)` is capability-dependent. Native validates it; do not infer that selecting channels for MFCO analysis disables other physical inputs. Preview.7 does not provide a general Python `enable_channel` method; use only the controls actually exported by your package.

For practical configuration code see `channel_setup.py` in the packaged Python examples and the shared setup in `examples/sdk/`. [Test pulses](2_6_test_signal.md) exercise the digital acquisition path and cannot validate analog threshold, hysteresis or edge response.

[Device operation](index.md) · [C channel controls](../07_c_api/7_7_channel_config.md)
