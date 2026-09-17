## End-to-End Firmware Field Update

The most advanced SDK workflow involves writing a new firmware image into a non-volatile flash slot and orchestrating the subsequent system reboot. The `field_update_e2e.py` script performs this safely by incorporating strict image validation, overwrite protection, and post-flash proof-of-life checks.

**Relevant script:**
*   `field_update_e2e.py`

### Workflow

1.  **Image Validation:** Supply a separately qualified `.bin` image for the target instrument. The loader accepts `<NAME>_<VERSION>.bin`, where `NAME` is four ASCII alphanumeric characters and `VERSION` is an unsigned decimal integer. `BOOT_001.bin` is an illustrative filename, not a preview.8 bundled image. Verify the supplier's checksum and target compatibility; there is no generic extended-filename/binary-header fallback in this workflow.
2.  **Handoff Orchestration:** Connect and inspect the protocol mode. From runtime, stop engines, request service entry and refresh field-update status so native performs service discovery. The [handoff tutorial](3_6_runtime_bootloader_handoff_validation.md) explains this sequence.
3.  **Slot Safety Validation:** The script refreshes the flash table and inspects the target slot state. By default, it will abort if the user attempts to overwrite a `VALID` slot or the designated default slot without explicit command-line override flags.
4.  **Image Streaming:** Call `device.load_field_update_image()`. Its synchronous progress callback supplies `NexatomFieldUpdateProgress`: phase, percentage, bytes and slot information. Phases include sending the image, programming and verifying; the [phase table](../01_getting_started/1_4_firmware.md#field-update-workflow) gives the public numeric values. Check the operation's result, not just its last progress percentage.
5.  **Post-Flash Configuration:** If requested, the script designates the newly flashed slot as the default boot partition using `device.set_field_update_default_slot_with_status()`.
6.  **Boot & Runtime Proof:** Native `device.boot_field_update_slot()` establishes runtime readiness on the same handle. The example additionally closes/reopens and, unless skipped, requests a brief CPS/telemetry data check. That diagnostic reconnect is separate from the native boot operation. Inspect the resulting profile to establish the active device/application contract.

### Execution

To run the field update script, the safety flag `--i-understand-this-writes-firmware` must be explicitly provided.

```powershell
python python\examples\field_update_e2e.py `
  --home . `
  --image "firmware/BOOT_001.bin" `
  --slot 1 `
  --boot-after-load `
  --i-understand-this-writes-firmware
```

Substitute the supplied image and an appropriate reported slot before running. Add `--set-default-after-load` only if you intend to change persistent boot preference. Linux uses the same arguments on one line (or shell backslash continuation instead of PowerShell backticks).

**Illustrative progress:** the script prints numeric phase values and actual byte counts; these lines show their form rather than a measured result.

```text
Discovering NexatomTT devices.
Selected device: name=UTT810, serial=NTT-00000001, connection=FTDI:1.
Connecting to hardware and checking protocol mode.
Runtime firmware detected; requesting field-upgrade service entry.
Field-update status: slot_count=2 default_slot=0
  slot 0: state=VALID version=1 default=True name=0x00000000
  slot 1: state=EMPTY version=0 default=False name=0x00000000
Loading firmware/BOOT_001.bin into slot 1.
phase=5 percent=<progress> bytes=<sent>/<total> slot=1 ...
phase=6 percent=<progress> bytes=<sent>/<total> slot=1 ...
phase=7 percent=<progress> bytes=<sent>/<total> slot=1 ...
Booting slot 1.
Reconnecting after boot and running runtime proof of life.
CPS total=0 period_ms=1000
Telemetry seq=1 uptime_s=5
Runtime proof callbacks: cps=<count> telemetry=<count>
Firmware image loading workflow complete.
```

Keep power and USB connected while programming. The example refuses writes without `--i-understand-this-writes-firmware`, protects valid/pending slots unless `--allow-valid-slot-overwrite` is supplied, and separately protects the default slot. Review those choices before loading; a bootable old image is useful for recovery. This tutorial changes firmware, whereas the ordinary acquisition and `boot_runtime.py` examples do not.
