## Booting into Runtime from Bootloader

An instrument may be running a measurement image or waiting in firmware service when an application connects. This tutorial requests a usable runtime through the native API without loading firmware or changing its persistent default. The same call also supports original Zynq without a bootloader.

**Relevant script:**
*   `boot_runtime.py`

### Workflow

The `boot_runtime.py` script uses `open_runtime_device()`. Its normal path delegates discovery, slot selection and readiness to native `connect_runtime`, retaining one device handle throughout.

1.  **Device Discovery:** The script invokes `lib.discover_devices(max_devices)` to locate attached UTT810 hardware.
2.  **Configuration:** `RuntimeBootOptions.timeout_ms` sets the native entry budget; allow 20000 ms for a cold entry. The preferred-slot option is for deliberate existing-image selection from service, not normal model/protocol setup.
3.  **Orchestration via Context Manager:** The `open_runtime_device()` context manager takes ownership of the boot sequence:
    *   **Existing Runtime:** Native attaches to and initializes the supported runtime without inventing a bootloader requirement.
    *   **Slot Selection:** When service must boot an image, normal native entry chooses a valid default slot, otherwise the lowest valid slot. An explicit `--boot-slot` selects a valid slot in the advanced service path.
    *   **Readiness:** Native completes the boot/runtime transition on the same handle. The application does not implement a bus rediscovery loop or interpret a boot ACK as readiness.
4.  **Yield to Runtime:** The context yields the ready `NexatomDevice`. Read its profile before configuring optional features or channel limits.

> **Failure handling.** No usable slot or an unsupported runtime causes an exception (`NexatomError` from native entry, or `RuntimeBootError` from helper selection). The helper does not silently program firmware. Inspect the error and use the [field-update workflow](3_7_end_to_end_firmware_field_update.md) only when an intentional image load is needed.

### Execution

To run the orchestration script, execute it from the command line. An explicit boot slot can optionally be provided.

```powershell
# Boot the default runtime slot
python python/examples/boot_runtime.py --home . --timeout-ms 20000

# Select slot 1 when connecting from service (if VALID)
python python/examples/boot_runtime.py --home . --boot-slot 1 --timeout-ms 20000
```

**Illustrative output** (device names and ports vary; on Linux the `connection_id` looks like `usb:2-1.3`):

```text
Discovering NexatomTT devices.
Selected device: name=UTT810, connection_id=usb:PCIROOT(0)#PCI(0801)#PCI(0004)#USBROOT(0)#USB(4), FT601 serial=000000000001 (information only).
Runtime firmware is ready.
```

If a runtime is already active, the helper attaches to it even when a preferred slot was supplied. Deliberately replacing the active image first requires the service workflow. After this tutorial, the [processed acquisition example](../01_getting_started/1_2_quick_start.md#configure-channels-and-record-processed-results) performs the same runtime entry automatically before recording.
