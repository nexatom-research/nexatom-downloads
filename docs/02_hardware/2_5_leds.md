## LEDs

The UTT810 features dedicated indicator LEDs for each input channel and a primary RGB status LED on the front panel.

> **Python Wrapper Support.** LED control is currently only exposed in the native C API. It is not yet wrapped in the `NexatomDevice` Python class.

### [Per-channel LED control](2_5_leds.md#per-channel-led-control)

Each of the 8 input channels has an associated indicator LED. By default, these LEDs may illuminate when a valid signal threshold is crossed. This behaviour can be explicitly overridden or disabled.

#### C API

```c
nexatom_tt_enable_channel_led(
    nexatom_tt_handle device,
    uint8_t channel,
    bool enable
);
```

| Parameter | Description |
|---|---|
| `channel` | Channel index (0–7) |
| `enable` | `true` to enable the indicator LED, `false` to disable |

### [RGB LED mode selection](2_5_leds.md#rgb-led-mode-selection)

The primary RGB status LED conveys overall device health, bootloader state, and protocol mode. Advanced users can override this automatic behavior by setting a custom LED mode value.

#### C API

```c
nexatom_tt_set_rgb_led_mode(
    nexatom_tt_handle device,
    uint32_t mode
);
```

The 32-bit `mode` parameter configures the color and blink pattern of the RGB LED. Mode mappings are strictly hardware-dependent.
