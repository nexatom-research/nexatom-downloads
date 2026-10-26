# Pulse Outputs (UTT160810)

The UTT160810 has eight pulse outputs (DTC outputs 0–7). Each output sends one pulse of a set width a set delay after a trigger. The trigger is an input of the board or the output's own pulse generator. The UTT810 has no pulse outputs; on it every call on this page returns `NEXATOM_ERROR_NOT_SUPPORTED`.

Pulse outputs are not part of any recording. Acquisition, file saving and calibration do not stop them, and they never stop a recording.

<a id="before-you-start"></a>

### [Before you start](2_13_dtc_pulse_outputs.md#before-you-start)

- **Firmware.** Use the current image `K168_005` (see [Firmware catalogues](../01_getting_started/1_4_firmware.md#firmware-catalogues)). The SDK knows the output latency of this image and states it in the device profile (`dtc_output_latency_ps`). On an image for which the profile reads 0, switching an output on, or switching the outputs on globally, returns `NEXATOM_ERROR_NOT_SUPPORTED` with the message "DTC output latency not available for this firmware (...)" and sends nothing. Switching an output off is always allowed.
- **Output type.** An apply needs the processed output type, `set_output_type(NEXATOM_OUTPUT_REALTIME_DATA)`.
- **Output 7 and the reference clock.** On the UTT160810 the reference clock output (REF OUT) uses the same SMA connector as output 7. While the 10 MHz reference clock output is on, scheduling output 7 returns `NEXATOM_ERROR_NOT_SUPPORTED`; outputs 0–6 are unaffected. See [External clock](2_8_external_clock.md#reference-clock-output).

<a id="fixed-output-latency"></a>

### [Fixed output latency](2_13_dtc_pulse_outputs.md#fixed-output-latency)

A pulse leaves the output a fixed time after its trigger, on top of the delay you program:

```
output edge = trigger time + dtc_output_latency_ps + delay_ps
```

`dtc_output_latency_ps` is in the device profile and is **180000 ps (180 ns)** for `K168_005`. It is the same for every output and every delay, and it does not depend on the trigger phase. It is known as soon as the connection has identified the image; no measurement needs to run. The SDK also checks it against the board's configuration read-back: if the board reports another value, the profile value becomes 0 for the rest of the connection, the outputs cannot be switched on, and the SDK logs one line naming both values.

Because of this latency, an output pulse can never come earlier than 180 ns after its trigger. To place a pulse at a time *T* after the trigger, program `delay_ps = T − dtc_output_latency_ps`; for *T* below the latency, delay the trigger signal itself, or use a later trigger.

```python
profile = device.get_device_profile()
latency_ps = int(profile.dtc_output_latency_ps)     # 180000 on K168_005
if latency_ps == 0:
    raise RuntimeError("pulse outputs are not available with this firmware")
wanted_ps = 250_000                                 # pulse 250 ns after the trigger
delay_ps = wanted_ps - latency_ps                   # 70 000 ps
```

<a id="applying-an-output"></a>

### [Applying an output](2_13_dtc_pulse_outputs.md#applying-an-output)

One call sets an output completely: `apply_dtc_output(configuration, timeout_ms=1000)` (C: `nexatom_tt_apply_dtc_output_v1()`). The configuration (`NexatomDtcOutputConfigV1`, C `nexatom_tt_dtc_output_config_v1_t`) holds:

| Field | Meaning |
|---|---|
| `output_id` | Output 0–7 |
| `source_channel` | Trigger input (an input allowed by the effective public input mask), or `NEXATOM_TT_DTC_SOURCE_PULSE_GENERATOR` (`0x80`) for the output's own pulse generator |
| `delay_ps` | Programmed delay, an exact multiple of 100 ps |
| `width_ps` | Pulse width, an exact multiple of 100 ps, at least 2000 ps |
| `enable` | 1 = on, 0 = off |

Then switch the outputs on globally with `set_dtc_global_enable(True)` (C: `nexatom_tt_set_dtc_global_enable_v1()`). An output sends pulses only while it is applied with `enable = 1` and the global enable is on. Acquisition may keep running while you apply.

The outcome of an apply:

| Result | Meaning |
|---|---|
| `NEXATOM_SUCCESS`, flags `VALID` and `ACCEPTED` | The board accepted the setting and it is in effect. If pulses of the previous setting were still on their way, the new setting takes over once they have left (the call waits for that within `timeout_ms`). |
| `NEXATOM_SUCCESS`, flags `VALID`, `ACCEPTED` and `PENDING` | Accepted, but older pulses were still on their way at `timeout_ms`. The setting takes effect by itself, at most the output's delay plus width later. |
| `NEXATOM_SUCCESS`, flag `PENDING` only | The board could not take the setting now (for example during calibration or another update). The SDK keeps it and applies it as soon as the board is ready. A later apply of the same output replaces it. |
| `NEXATOM_ERROR_CONFIGURATION_FAILED` | The board refused the setting; the result's reason says why. In Python the call raises `NexatomDtcApplyRejected`, whose `result` holds the record. |

Outputs never wait for each other.

<a id="output-limits"></a>

#### Output limits

`get_dtc_output_limits(output)` (C: `nexatom_tt_get_dtc_output_limits()`) returns the range an output accepts, without waiting and without asking the board:

- the smallest width, `min_width_ps`: 2000 ps on every output;
- `max_delay_plus_width_ps`: the largest `delay_ps + width_ps` (about 8.59 s). The largest delay for a width is `max_delay_plus_width_ps − width_ps`. A pair past it is refused before anything is sent, with `NEXATOM_ERROR_CONFIGURATION_FAILED` and the reason `RESET_OVERFLOW`.

Take a user interface's limits from this call. These limits do not depend on the trigger rate. Whether pulses stay separate at a given rate is the merge rule below.

<a id="pulses-that-come-too-close"></a>

### [Pulses that come too close: the merge rule](2_13_dtc_pulse_outputs.md#pulses-that-come-too-close)

Each output can hold up to **8 pulses** at a time. A pulse holds its place from its trigger until it has left the output, that is for `delay_ps + width_ps + K`, where *K* is 8 to 12 ns depending on the output. The fixed output latency holds no place.

**Every trigger gives its own pulse while**

```
trigger rate × (delay_ps + width_ps + K) < 8
```

For a periodic trigger with period *T*, the largest delay that keeps every pulse separate is

```
largest separate delay = 8 × T − width_ps − K
```

Use *K* = 16 ns for a bound that fits every output.

| Trigger | Width | Pulses stay separate up to a delay of about |
|---|---|---|
| *T* = 8 ns (125 MHz) | 2 ns | 50 ns |
| *T* = 100 ns (10 MHz) | 10 ns | 780 ns |

The width must also stay below the trigger period.

**What happens past the limit.** When a pulse would need a ninth place, or would start at or before the end of the pulse before it, the trigger is not dropped: it lengthens the pulse before it to its own end. A periodic trigger past the limit then gives, in each group, 7 normal pulses and 1 long one (`width_ps + n × T`). A delay far past the limit keeps the output high most of the time. Outputs never affect each other.

Plan the delay and width with the rule above. An application only needs to tell its user that pulses joined; `get_dtc_output_limits()` does not include this rule.

<a id="pulse-generators"></a>

### [Pulse generators](2_13_dtc_pulse_outputs.md#pulse-generators)

Each output has its own periodic pulse generator. To drive output *k* from it:

1. `set_dtc_pulse_generator_period(k, period_ps)`: any whole number of ps from 8000 ps (125 MHz) to 2^40 − 1 ps (about 1.1 s); 1 µs after power-up.
2. `apply_dtc_output()` for output *k* with `source_channel = NEXATOM_TT_DTC_SOURCE_PULSE_GENERATOR`, `enable = 1` and the delay and width you want. They mean exactly what they mean for an input trigger.
3. `set_dtc_global_enable(True)`.
4. `set_dtc_pulse_generator_enable_mask(mask)` with bit *k* set.

Pulse *n* of a generator switched on at time *t*<sub>start</sub> leaves the output at *t*<sub>start</sub> + *n* × `period_ps` + `delay_ps` + the fixed output latency, within ±50 ps. The mean period is exact over any run length. *t*<sub>start</sub> is not related to any input time tag.

- **One mask write starts generators together.** Generators switched on by the same mask write start on the same clock edge, so outputs with equal periods stay locked and their delays set the offsets between them. To add a generator without disturbing the running ones, pass the old mask plus its bit. A generator whose bit is cleared stops at once; a pulse it had started still finishes.
- **Changing a period.** A running generator takes a new period from its next pulse without restarting its phase. Two outputs changed by two calls are not guaranteed to change at the same pulse; to lock them again, switch them off and on in one mask write.
- **Rates.** The merge rule applies as for an input trigger. At an 8 ns period, pulses stay separate only for widths up to 4 ns.
- **Getters.** `get_dtc_pulse_generator_period(output)` and `get_dtc_pulse_generator_enable_mask()` return the values this session last wrote; the board has no read-back of them. After connect the mask reads 0.

These calls need `NEXATOM_TT_FEATURE_DTC_PULSE_GENERATOR` in the profile; otherwise they return `NEXATOM_ERROR_NOT_SUPPORTED` and write nothing.

<a id="automatic-re-apply"></a>

### [Automatic re-apply and restore](2_13_dtc_pulse_outputs.md#automatic-re-apply)

The SDK remembers, for each output, the last setting the board accepted, and applies it again by itself when the board drops it:

- after `set_channel_input_delay()` changes the delay of the output's trigger input (writing the value the board already has changes nothing);
- after `restart_dtc_calibration()`, once the outputs are ready again (waited for at most 10 s);
- when the board drops a setting for any other reason, once. A second drop of the same output without any action by the application is reported (WARN log line, `second_drop_mask`) and not applied again.

After a measurement abort the board restarts its runtime (see [Telemetry](2_11_telemetry.md#measurement-abort)). The SDK then applies every remembered output again and, if this session had the outputs switched on globally, switches the system and the global enable back on and writes the application's last pulse generator mask once more.

An apply or disable by the application replaces the remembered setting (a disable forgets it). A setting the board refuses on re-apply is not changed or clamped: the output stays off until the application applies or disables it. `get_dtc_reapply_status()` (C: `nexatom_tt_get_dtc_reapply_status()`) shows which outputs are remembered, pending, refused or failed, and the last restore after a runtime restart. It reads the SDK's own state and sends nothing. Each re-apply logs one INFO line, each refusal or failure one WARN line.

Disconnect forgets every remembered setting and switches every pulse generator off.

<a id="dtc-status"></a>

### [Status and calibration](2_13_dtc_pulse_outputs.md#dtc-status)

`get_dtc_status(timeout_ms=1000)` (C: `nexatom_tt_get_dtc_status_v1()`) reads one fresh status snapshot of the outputs, including the outputs whose setting is still waiting (`apply_pending_mask`) and the result of the last output calibration. It needs the processed output type. `restart_dtc_calibration()` restarts the output calibration; it returns once the request is sent, and the remembered settings are applied again afterwards as described above. It needs `NEXATOM_TT_FEATURE_DTC_CALIBRATION_RESTART` in the profile; completion is not returned, so watch `get_dtc_status()`.

The complete example `python/examples/utt_16_8_fast_dtc.py`, given its write flag, selects the processed output type itself, applies an output, switches the outputs on and reads the status.
