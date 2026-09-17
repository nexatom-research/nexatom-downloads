## LEDs

The SDK provides controls for channel indicators and the device's RGB indicator where the supplied hardware implements them. The physical position, wiring and automatic display patterns are board-dependent; these controls do not replace reading protocol/readiness status from the API.

Both control groups are available in the C API and Python `NexatomDevice` class.

### Per-channel LED control

An input channel's indicator can be enabled or disabled. Use a channel present in the resolved profile's `effective_public_tdc_mask`. An illuminated LED is not evidence that the host has received or saved a time tag.

#### C API

```c
nexatom_error_code_t nexatom_tt_enable_channel_led(
    nexatom_tt_handle device,
    uint8_t channel,
    bool enable
);
```

| Parameter | Description |
|---|---|
| `channel` | Channel index (0–7) |
| `enable` | `true` to enable the indicator LED, `false` to disable |

#### Python

```python
device.enable_channel_led(channel=0, enable=True)
```

### RGB LED mode selection

The RGB control accepts a mode value defined by the particular hardware image. Use the native profile/protocol status to determine bootloader or runtime readiness; a color alone is not a cross-model status contract.

#### C API

```c
nexatom_error_code_t nexatom_tt_set_rgb_led_mode(
    nexatom_tt_handle device,
    uint32_t mode
);
```

The 32-bit `mode` parameter selects the hardware-defined LED mode. There is no common public color/blink lookup table in preview.8. In Python the equivalent method is `device.set_rgb_led_mode(mode)`; use a value documented for your supplied image rather than guessing one.
