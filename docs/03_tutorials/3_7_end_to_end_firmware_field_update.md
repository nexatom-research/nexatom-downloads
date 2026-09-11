## End-to-End Firmware Field Update

The most advanced SDK workflow involves writing a new firmware image into a non-volatile flash slot and orchestrating the subsequent system reboot. The `field_update_e2e.py` script performs this safely by incorporating strict image validation, overwrite protection, and post-flash proof-of-life checks.

**Relevant script:**
*   `field_update_e2e.py`

### Workflow

1.  **Image Validation:** The script verifies that the provided `.bin` file exists and strictly adheres to the UTT810 firmware naming convention (e.g., `nexatomtt_fwp_utt810_*.bin`).
2.  **Handoff Orchestration:** The script establishes a connection and checks the current protocol mode. If the device is actively acquiring data (`RUNTIME`), it halts the hardware engines and initiates the field-upgrade service entry (as detailed in Section 3.6), polling until `BOOTLOADER` mode is achieved.
3.  **Slot Safety Validation:** The script refreshes the flash table and inspects the target slot state. By default, it will abort if the user attempts to overwrite a `VALID` slot or the designated default slot without explicit command-line override flags.
4.  **Image Streaming:** The host calls `device.load_field_update_image()`. During execution, the native thread invokes a registered Python callback passing `NexatomFieldUpdateProgress` structures. This provides granular, realtime visibility into the state machine phases (`ERASE`, `WRITE`, `VERIFY`, and `FINALIZE`).
5.  **Post-Flash Configuration:** If requested, the script designates the newly flashed slot as the default boot partition using `device.set_field_update_default_slot_with_status()`.
6.  **Boot & Runtime Proof:** The script triggers `device.boot_field_update_slot()`, catches the USB bus reset, and re-enumerates the hardware via its unique `DeviceIdentity`. Once `RUNTIME` mode is confirmed, it performs a brief "proof of life" test by verifying the new image actively emits valid CPS and telemetry packets.

### Execution

To run the field update script, the safety flag `--i-understand-this-writes-firmware` must be explicitly provided.

```powershell
python python\examples\field_update_e2e.py `
  --image "nexatomtt_fwp_utt810_v1.0.0_20240401.bin" `
  --slot 1 `
  --set-default-after-load `
  --boot-after-load `
  --i-understand-this-writes-firmware
```

**Expected Output:**

```text
Discovering NexatomTT devices.
Selected device: name=UTT810, serial=NTT-00000001, connection=FTDI:1.
Connecting to hardware and checking protocol mode.
Runtime firmware detected; requesting field-upgrade service entry.
Field-update status: slot_count=2 default_slot=0
  slot 0: state=VALID version=1 default=True name=0x00000000
  slot 1: state=EMPTY version=0 default=False name=0x00000000
Loading nexatomtt_fwp_utt810_v0.1.0-preview.6_20240401.bin into slot 1.
Phase ERASE: 100.0% (134217728/134217728 bytes)
Phase WRITE: 100.0% (12845056/12845056 bytes)
Phase VERIFY: 100.0% (12845056/12845056 bytes)
Phase FINALIZE: 100.0% (256/256 bytes)
Setting slot 1 as the default slot.
Field-update status: slot_count=2 default_slot=1
Booting slot 1.
Reconnecting after boot and running runtime proof of life.
CPS total=0 period_ms=1000
Telemetry seq=1 uptime_s=5
Runtime proof of life succeeded. Field update E2E complete.
```
