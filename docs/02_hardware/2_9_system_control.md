## System Control

Global hardware state management—including data path enabling, peripheral resetting, and synchronized acquisition halting—is handled through the SDK's system control interfaces.

### [System enable / disable (Data Shuffler)](2_9_system_control.md#system-enable-disable)

The system enable function controls the primary data shuffler within the FPGA. When disabled, no time tags are processed and no data is forwarded to the active output mode.

> **Best Practice.** Always disable the system before performing bulk register configurations (e.g., threshold adjustments, edge type changes, or routing). Re-enable the system only after all configurations have been committed to avoid generating spurious time tags during the transition state.

#### C API

```c
nexatom_tt_enable_system(
    nexatom_tt_handle device,
    bool enable
);
```

#### Python

```python
# Suspend processing
device.enable_system(False)

# ... perform channel configuration ...

# Resume processing
device.enable_system(True)
```

### [Peripheral reset](2_9_system_control.md#peripheral-reset)

The peripheral reset function asserts or de-asserts a hardware reset line distributed to all peripheral sub-components within the FPGA logic.

To perform a complete reset cycle, the host must explicitly assert the reset state and subsequently de-assert it to return the device to normal operation.

#### C API

```c
nexatom_tt_reset_peripherals(
    nexatom_tt_handle device,
    bool reset
);
```

#### Python

```python
# Assert peripheral reset
device.reset_peripherals(True)

# Return to normal operation
device.reset_peripherals(False)
```

### [Global acquisition stop](2_9_system_control.md#global-acquisition-stop)

The global stop function pulses a synchronous stop signal to all active hardware acquisition engines (TIHI, MFCO, CORL, CORM). This serves as an immediate, software-triggered termination condition for any running histograms or correlations, ensuring all modules stop acquiring at the same deterministic clock edge.

#### C API

```c
nexatom_tt_request_global_stop_all_modes(nexatom_tt_handle device);
```

#### Python

```python
device.request_global_stop_all_modes()
```
