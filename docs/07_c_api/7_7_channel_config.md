# Channel Configuration

These endpoints configure authorized public inputs. Validate the native effective channel mask, features and limits first; channel numbers do not alone establish physical connector mapping.

Through this module, developers can independently tune the voltage discriminators, apply synthetic digital delays to align mismatched cable lengths, and inject simulated test signals into the measurement pipeline.

### Function Reference

| Function | Parameters (In/Out) | Returns | Description |
| :--- | :--- | :--- | :--- |
| `nexatom_tt_set_channel_threshold` | `[In] nexatom_tt_handle device`<br>`[In] uint8_t channel`<br>`[In] uint16_t threshold_mv` | `nexatom_error_code_t` | Sets the analog voltage comparator trigger level (in millivolts) for a specific physical input (e.g., `0` through `7`). |
| `nexatom_tt_set_channel_input_delay` | `[In] nexatom_tt_handle device`<br>`[In] uint8_t channel`<br>`[In] uint32_t delay_ps` | `nexatom_error_code_t` | Requests delay in ps up to the resolved `max_channel_input_delay_ps`; preview.7 is not restricted to the older universal 4000 ps assumption. |
| `nexatom_tt_set_channel_edge_type` | `[In] nexatom_tt_handle device`<br>`[In] uint8_t channel`<br>`[In] uint8_t edge_type` | `nexatom_error_code_t` | Sets the signal discriminator to trigger on a rising edge (`0`) or falling edge (`1`). |
| `nexatom_tt_set_channel_hysteresis` | `[In] nexatom_tt_handle device`<br>`[In] uint8_t channel`<br>`[In] uint16_t hysteresis_mv` | `nexatom_error_code_t` | Adjusts the analog comparator hysteresis window (0-175 mV) to reject high-frequency baseline electrical noise. |
| `nexatom_tt_set_channel_routing` | `[In] nexatom_tt_handle device`<br>`[In] uint8_t source_channel`<br>`[In] uint8_t target_channel` | `nexatom_error_code_t` | Routes events from `source_channel` to the logical `target_channel` in the FPGA routing matrix. Bypasses the default 1-to-1 physical-to-logical mapping. |
| `nexatom_tt_enable_channel_test_pulse`| `[In] nexatom_tt_handle device`<br>`[In] uint8_t channel`<br>`[In] bool enable` | `nexatom_error_code_t` | Routes the internal FPGA oscillator to the digital input, allowing logical verification of processing algorithms without external physical signals. |
| `nexatom_tt_set_channel_test_pulse_params`| `[In] nexatom_tt_handle device`<br>`[In] uint8_t channel`<br>`[In] uint32_t period_cycles`<br>`[In] uint32_t width_cycles`| `nexatom_error_code_t` | Period/width in the profile's test-pulse clock cycles; validate reported bounds and `0 < width < period`. |
| `nexatom_tt_enable_channel_led` | `[In] nexatom_tt_handle device`<br>`[In] uint8_t channel`<br>`[In] bool enable` | `nexatom_error_code_t` | Overrides the hardware "Signal Detected" LED indicator on the front panel for a specific channel. |
| `nexatom_tt_set_rgb_led_mode` | `[In] nexatom_tt_handle device`<br>`[In] uint32_t mode`| `nexatom_error_code_t` | Overrides the master RGB status LED pattern for custom application signaling (e.g., solid green, blinking red). |

### C Example: Aligning Cables and Noise Rejection

```c
// Assume Channel 0 is our reference pulse and Channel 1 is our photon detector
uint8_t ch_sync = 0;
uint8_t ch_detector = 1;

// Fragment: validate these illustrative values against profile and input source.
nexatom_tt_set_channel_threshold(my_device, ch_sync, 1500); // 1.5V
nexatom_tt_set_channel_threshold(my_device, ch_detector, 800); // 0.8V

// Request hysteresis; acceptance does not prove analog noise rejection.
nexatom_tt_set_channel_hysteresis(my_device, ch_detector, 50);

// Request delay after checking the profile limit; measure actual cable timing.
// Complete code must check every return and preserve cleanup on failure.
nexatom_tt_set_channel_input_delay(my_device, ch_detector, 1500);
```
