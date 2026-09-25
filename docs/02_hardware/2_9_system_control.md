## System Control

Global hardware state management—including data path enabling, peripheral resetting, and synchronized acquisition halting—is handled through the SDK's system control interfaces.

### System enable / disable (Data Shuffler)

The system enable function controls acquisition in the FPGA's primary data path. Disabling it prevents the intended measurement from running; control traffic and diagnostic responses have their own paths. Set `NO_OUTPUT` as well when preparing a quiet transition or changing file-saving configuration.

> **Best Practice.** Always disable the system before performing bulk register configurations (e.g., threshold adjustments, edge type changes, or routing). Re-enable the system only after all configurations have been committed to avoid generating spurious time tags during the transition state.

#### C API

```c
nexatom_error_code_t nexatom_tt_enable_system(
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

### Peripheral reset

The peripheral reset function asserts or de-asserts the FPGA peripheral reset control. It is a destructive measurement transition: stop active measurements and complete saver cleanup before using it. Routine runtime connection does not require a user-written reset/initialization sequence; `connect_runtime()` owns startup.

To perform a complete reset cycle, the host must explicitly assert the reset state and subsequently de-assert it to return the device to normal operation.

#### C API

```c
nexatom_error_code_t nexatom_tt_reset_peripherals(
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

Successful reset release (`nexatom_tt_reset_peripherals(device, false)`) also selects `REALTIME_DATA` in the native implementation, as a pipeline barrier after the reset. Explicitly select the output required by the next operation after a deliberate reset; do not assume a previous raw/quiet output mode survived it. A peripheral reset is not the service-entry operation used for firmware updates.

### Global acquisition stop

The global stop function requests a shared stop for active hardware measurement engines such as TIHI, MFCO and correlation. It is useful when ending an experiment with several engines enabled. A successful host call confirms command admission; it does not prove that all terminal packets have already reached the host or that files are closed.

#### C API

```c
nexatom_error_code_t nexatom_tt_request_global_stop_all_modes(nexatom_tt_handle device);
```

#### Python

```python
device.request_global_stop_all_modes()
```

Retain callbacks/savers long enough to receive the terminal data required by the experiment. Then set `NO_OUTPUT`, disable the system and test sources, close active savers, and disconnect. Attempt every cleanup step even if an earlier one fails, while preserving the errors. The SDK examples demonstrate this sequence without forcing a process exit or closing the FTDI handle behind the native library.
