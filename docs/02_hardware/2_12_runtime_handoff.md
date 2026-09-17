# 2.12 Runtime handoff

For measurements use native `connect_runtime`; for deliberate firmware service use the public handoff operations. Keep the same device handle and its callbacks through supported service/boot transitions.

`request_field_upgrade_service_entry()` requests firmware service. Refresh its status and slot table before selecting an image. `boot_field_update_slot` boots an existing valid slot and waits for native runtime readiness. The normal startup helper can choose an existing valid slot without rewriting firmware.

Do not implement a USB detach/re-enumeration loop, assume the serial changes, or call an EEPROM programming utility. Entering resident firmware service is different from changing persistent USB configuration or the default boot slot.

An explicit preferred slot in Python startup selects the service-mode boot target; it does not forcibly replace an already-running runtime. For a deliberate round trip, use the [handoff tutorial](../03_tutorials/3_6_runtime_bootloader_handoff_validation.md).

Check returned errors and the resulting identity/profile. UNKNOWN protocol, timeout and failed readiness are failures, not reasons to issue controls optimistically.

[Device operation](index.md) · [Startup guide](../06_in_depth_guides/6_6_bootloader_first_device_startup.md)
