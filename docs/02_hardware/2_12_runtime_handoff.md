## Runtime / Bootloader Handoff

Bootloader-equipped instruments have two firmware environments: `RUNTIME` for acquisition and measurement, and `BOOTLOADER` for flash slot management and updates. Original Zynq instruments without a bootloader have runtime only. The same native API discovers the attachment, resolves its profile and owns the required startup/handoff sequence; a GUI, C program or Python script is a client of that lifecycle.

### Service entry for field upgrades

If a bootloader-equipped device is running an application image, request service entry before a field update. This normally does not require a power cycle. A device known to have no bootloader returns `NEXATOM_ERROR_NOT_SUPPORTED` without sending the service-entry control.

#### C API

```c
/* Request entry; check the result, then confirm service through status. */
nexatom_error_code_t rc = nexatom_tt_request_field_upgrade_service_entry(device);
/* Use nexatom_tt_refresh_field_update_status() to obtain confirmed service
   status/slots before an update. See the field-update reference for buffers. */
```

#### Python

```python
device.request_field_upgrade_service_entry()
status, slots = device.refresh_field_update_status()
# A returned service status/slot table is the evidence for the next operation.
```

The native request quiets normal runtime traffic, transfers transport ownership to the service path and sends the appropriate runtime handoff command. **Request success alone does not prove that firmware reset or entered service.** Confirm service using the response/status API before proceeding. Zynq PS and Kintex MicroBlaze implement the hardware transition differently; clients do not need separate handoff protocols or a USB disconnect/re-enumeration loop.

If aborting a pending request, use `clear_field_upgrade_service_entry_request()` (`nexatom_tt_clear_field_upgrade_service_entry_request` in C). Do not call it immediately after every entry request: it is a cancellation/recovery operation. When service is already confirmed, clearing the request is a native host-state no-op and does not send runtime commands to the bootloader.

### Bootloader-first boot orchestration

Because an instrument may power on in runtime or service, the native `connect_runtime()` operation performs the common measurement startup. It leaves an already usable runtime in its current acquisition/output mode, or boots an existing valid image when needed, and returns success only after decoded runtime identity and control authorization are established. It never loads firmware or changes the stored default slot.

#### C API

```c
/* device is the handle created from the selected discovery result. */
nexatom_error_code_t rc = nexatom_tt_connect_runtime(device, 20000);
if (rc != NEXATOM_SUCCESS) {
    /* Report the actual error; do not blindly resend a boot command. */
}
/* Only configure acquisition after success. */
```

In Python, call `device.connect_runtime(timeout_ms=20000)` on an existing device, or use `open_runtime_device()` from `nexatomtt.runtime_boot` to manage handle lifetime as well:

```python
from nexatomtt.runtime_boot import RuntimeBootOptions, open_runtime_device

# library and device_info come from discovery of the selected instrument.
with open_runtime_device(library, device_info,
                         RuntimeBootOptions(timeout_ms=20000)) as device:
    profile = device.get_device_profile()
    # Configure channels, register callbacks/open savers, then acquire.
    # Stop output and close savers before leaving the context.
```

#### Configuration (`RuntimeBootOptions`)

```python
from nexatomtt.runtime_boot import RuntimeBootOptions

options = RuntimeBootOptions(
    timeout_ms=20000,          # Native connection + boot/readiness budget
    mode_timeout_sec=20.0,     # Used only for explicit preferred-slot inspection
    poll_sec=0.25,             # Used only for that explicit mode-inspection loop
    preferred_slot=None       # Normal measurement: native chooses existing image
)
```

#### Slot selection logic (`select_runtime_slot()`)

Normal runtime entry chooses the valid stored default slot, otherwise the lowest-indexed valid slot. An optional explicit `preferred_slot` is available for a field-update/verification workflow and is checked by the helper's `select_runtime_slot()` function:

1.  **Preferred Slot:** The slot specified in `RuntimeBootOptions.preferred_slot`, provided its state is `VALID`.
2.  **Default Slot:** The slot marked as `is_default` in the bootloader's flash table, provided its state is `VALID`.
3.  **Fallback:** The lowest-indexed slot whose state is `VALID`.

A requested slot must be `VALID`; a missing/invalid preferred slot is rejected. Normal native runtime entry returns `NEXATOM_ERROR_NOT_SUPPORTED` if no valid slot is available (surfaced as `NexatomError` in Python). The helper's explicit-slot selection can raise `RuntimeBootError`. Neither failure authorizes silently loading an image or changing which persistent slot is default.

### Post-boot USB re-enumeration and device identity

The explicit `nexatom_tt_boot_field_update_slot()` / `device.boot_field_update_slot()` operation is synchronous with native runtime startup: success requires usable decoded runtime identity and controls. Retain the same handle and registered callbacks. Do not recreate it, issue another peripheral reset or repeat startup register writes after success.

The FTDI bridge may remain enumerated throughout the firmware transition. The following identity information remains useful to applications, but the normal startup path does not require a Python-owned USB reconnect loop:

| Field | Description |
|---|---|
| `connection_id` | Transport-specific connection identifier; not a permanent board identity |
| `serial_number` | USB bridge serial string, separate from the runtime image |
| `product_model_id` | Native profile's model identity and associated contract |
| `application_image_id` | Running image identity; changing firmware does not require changing the USB serial |

A boot ACK is not runtime readiness. If an explicit boot times out after transmission, preserve the error/status evidence and inspect the device; do not automatically send a second boot request. Run blocking boot/service operations off an application's UI thread. The ordinary workflow uses one instrument at a time, and both C and Python delegate these lifecycle decisions to the same native library.
