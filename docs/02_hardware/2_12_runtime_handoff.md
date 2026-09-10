## Runtime / Bootloader Handoff

The UTT810 operates in one of two mutually exclusive firmware environments: `RUNTIME` (where data acquisition and measurement occur) or `BOOTLOADER` (where flash slot management and firmware updates occur). Navigating between these states safely is governed by the handoff API.

### [Service entry for field upgrades](2_12_runtime_handoff.md#service-entry-for-field-upgrades)

If the device is currently executing runtime firmware, it must be explicitly instructed to reboot into the bootloader before a field update can commence. This is accomplished by issuing a service entry request.

#### C API

```c
/* Request bootloader entry */
nexatom_tt_request_field_upgrade_service_entry(device);

/* Clear a sticky service entry request if aborted */
nexatom_tt_clear_field_upgrade_service_entry_request(device);
```

#### Python

```python
device.request_field_upgrade_service_entry()
device.clear_field_upgrade_service_entry_request()
```

Upon receiving a service entry request, the runtime firmware halts data acquisition, flushes pending queues, sets a handoff bit in memory, and triggers a system reset. The host software must then await USB re-enumeration to reconnect and verify `BOOTLOADER` protocol mode.

### [Bootloader-first boot orchestration](2_12_runtime_handoff.md#bootloader-first-boot-orchestration)

Because the UTT810 may power on in either mode (depending on default slot settings and previous state), host applications must robustly orchestrate the boot sequence to ensure the device is in `RUNTIME` mode before commencing acquisition.

*(Note: While the Python SDK abstracts the boot orchestration, C applications must manually implement the USB polling loop using `nexatom_tt_boot_field_update_slot()`.)*

In the Python SDK, this sequence is abstracted by the `open_runtime_device()` context manager located in the `nexatomtt.runtime_boot` module.

#### Configuration (`RuntimeBootOptions`)

```python
from nexatomtt.runtime_boot import RuntimeBootOptions

options = RuntimeBootOptions(
    timeout_ms=5000,           # USB connection timeout
    mode_timeout_sec=20.0,     # Max wait for protocol mode resolution
    poll_sec=0.25,             # USB polling interval during reconnect
    preferred_slot=None        # Optional: override default boot slot
)
```

#### Slot selection logic (`select_runtime_slot()`)

If the device powers on in `BOOTLOADER` mode, `open_runtime_device()` automatically requests the flash slot table and determines the best image to boot using the following strict precedence:

1.  **Preferred Slot:** The slot specified in `RuntimeBootOptions.preferred_slot`, provided its state is `VALID`.
2.  **Default Slot:** The slot marked as `is_default` in the bootloader's flash table, provided its state is `VALID`.
3.  **Fallback:** The lowest-indexed slot whose state is `VALID`.

If no `VALID` slots exist, the process aborts with a `RuntimeBootError`.

### [Post-boot USB re-enumeration and device identity](2_12_runtime_handoff.md#post-boot-usb-re-enumeration-and-device-identity)

Issuing a boot command (`nexatom_tt_boot_field_update_slot()`) causes the FPGA to trigger a physical USB disconnect event, transitioning from the bootloader to the runtime firmware image.

To ensure the host reconnects to the exact same physical hardware in multi-device setups, the `open_runtime_device()` orchestrator captures a `DeviceIdentity` snapshot prior to the reboot:

| Field | Description |
|---|---|
| `connection_id` | The physical USB port path and FTDI index |
| `serial_number` | The hardware-burned serial number string |

Upon USB re-enumeration, the orchestrator loops `discover_devices()` and filters the list against the captured `connection_id` and `serial_number` to re-establish the `NexatomDevice` handle seamlessly.
