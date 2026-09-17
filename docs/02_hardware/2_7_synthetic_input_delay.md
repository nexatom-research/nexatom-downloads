# 2.7 Synthetic input delay

## Per-channel input delay configuration

`set_channel_input_delay(channel, delay_ps)` and `nexatom_tt_set_channel_input_delay` accept integer picoseconds. First validate the selected channel and `profile.max_channel_input_delay_ps`.

Preview.7 supports profile limits beyond the older manual's fixed 4000 ps range. Do not replace that old limit with a universal 256000 ps allowance: the connected profile remains authoritative, including a zero limit. The published Zynq checks include settings above 200 ns, but do not qualify every model/image or infer picosecond physical accuracy from API units.

```python
# Fragment inside a connected, authorized measurement setup; output is quiet.
profile = device.get_device_profile()
channel, requested_ps = 1, 220000
if not (int(profile.effective_public_tdc_mask) & (1 << channel)):
    raise ValueError("Channel is not authorized")
if requested_ps > int(profile.max_channel_input_delay_ps):
    raise ValueError("Requested delay exceeds this device profile")
device.set_channel_input_delay(channel, requested_ps)
# Acceptance records the requested control value, not analog readback.
```

[Device operation](index.md) · [Channel controls](../07_c_api/7_7_channel_config.md)
