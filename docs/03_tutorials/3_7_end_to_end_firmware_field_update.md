# 3.7 End-to-end firmware field update

Use this only with an image supplied for your instrument. Preview.7 includes no image. Inspect the intended device, reported slot table and supplier compatibility/checksum information before writing.

```sh
# Display safeguards without opening or writing a device.
python python/examples/field_update_e2e.py --help
# Example write command: replace the illustrative image and slot deliberately.
# This changes persistent firmware storage; keep USB and power connected.
python python/examples/field_update_e2e.py --home . --image /path/to/BOOT_001.bin --slot 1 --i-understand-this-writes-firmware --boot-after-load
```

The filename must follow the loader's four-alphanumeric-character application ID plus decimal version convention. A valid name does not establish hardware compatibility.

The tool enters/uses service, applies overwrite safeguards, loads the image and checks completion. `--allow-valid-slot-overwrite` and `--allow-default-slot-overwrite` are explicit overrides, not routine options. `--set-default-after-load` is a separate persistent choice and is deliberately absent above. Retain another known-good image where possible.

Use the actual progress record and final return status, not assumed phase names or a fixed 5–15 second duration. If boot is requested, native establishes readiness on the same handle; verify runtime identity and acquisition afterward. Do not report failure as successful merely because some data was written or progress reached a percentage.

[Tutorials](index.md) · [Firmware management](../01_getting_started/1_4_firmware.md)
