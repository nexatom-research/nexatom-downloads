# 2.5 LEDs

## Per-channel LED control

The Python method `enable_channel_led(channel, enable)` calls the corresponding public C API. Validate the channel/profile first and handle an unsupported operation normally.

## RGB LED mode selection

`set_rgb_led_mode(mode)` selects a firmware-supported indicator mode. Use the definitions supplied for the instrument's firmware. This manual does not assign unverified colours, blink patterns or electrical meaning to every model.

An LED change is not proof that acquisition, clock lock or calibration has completed. Use the relevant public status/results instead.

[Device operation](index.md) · [Channel configuration API](../07_c_api/7_7_channel_config.md)
