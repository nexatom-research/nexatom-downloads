## Booting into Runtime from Bootloader

When powered on, UTT810 devices typically start in the bootloader's boot-decision window. To perform data acquisition, host software must orchestrate a transition to the runtime firmware image. This tutorial demonstrates how to execute this handoff seamlessly without writing or modifying firmware.

**Relevant script:**
*   `boot_runtime.py`

### Workflow

The `boot_runtime.py` script relies on the `open_runtime_device()` context manager to abstract the complexities of USB re-enumeration and slot selection.

1.  **Device Discovery:** The script invokes `lib.discover_devices(max_devices)` to locate attached UTT810 hardware.
2.  **Configuration:** A `RuntimeBootOptions` dataclass is instantiated to define the USB connection timeout, mode resolution timeout, polling interval, and an optional preferred boot slot.
3.  **Orchestration via Context Manager:** The `open_runtime_device()` context manager takes ownership of the boot sequence:
    *   **Mode Detection:** Determines if the hardware is already in `RUNTIME` mode or currently in `BOOTLOADER` mode.
    *   **Slot Selection:** If in `BOOTLOADER` mode, it requests the flash slot table and determines the best `VALID` image based on strict precedence (preferred slot $\rightarrow$ default slot $\rightarrow$ lowest-indexed valid slot).
    *   **Reboot and Reconnect:** It issues the boot command, catches the resulting hardware USB disconnect, and polls the bus to re-acquire the device using its cached physical `connection_id` and `serial_number`.
4.  **Yield to Runtime:** Once the device re-enumerates and confirms `RUNTIME` protocol mode, the context manager yields the `NexatomDevice` handle.

> **Note.** If all flash slots are `EMPTY` or `CORRUPT`, the context manager will abort and raise a `RuntimeBootError`. In this scenario, firmware must be programmed using the field update workflow (see Section 3.7).

### Execution

To run the orchestration script, execute it from the command line. An explicit boot slot can optionally be provided.

```powershell
# Boot the default runtime slot
python python\examples\boot_runtime.py

# Force boot into slot 1 (if VALID)
python python\examples\boot_runtime.py --boot-slot 1
```

**Expected Output:**

```text
Discovering NexatomTT devices.
Selected device: name=UTT810, serial=NTT-00000001, connection=FTDI:1.
Runtime firmware is ready.
```
