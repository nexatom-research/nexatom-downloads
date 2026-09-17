# 3.2 Booting into runtime from bootloader

```sh
# Request a usable measurement runtime, including boot from an existing valid slot.
python python/examples/boot_runtime.py --home . --timeout-ms 20000
```

The normal path calls native `connect_runtime` on one handle. Native identifies the active protocol, selects an existing valid runtime when in service, boots it and waits for profile/control readiness. The client does not need USB re-enumeration or model-specific delays.

If an explicit service-mode boot target is needed, use the example's `--boot-slot` option after checking the slot table. That option does not force replacement of an already-running runtime. Use the [service round-trip tutorial](3_6_runtime_bootloader_handoff_validation.md) for that deliberate operation.

No image is written and the default slot need not change. If no valid image is available, stop and obtain a compatible image through the [firmware workflow](3_7_end_to_end_firmware_field_update.md). A successful connection does not itself start your intended acquisition.

[Tutorials](index.md) · [Startup helper](../06_in_depth_guides/6_6_bootloader_first_device_startup.md)
