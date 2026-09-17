# 3.6 Runtime/bootloader handoff validation

This example deliberately enters firmware service and can boot an existing valid image back into runtime. It writes no image and does not require changing the default slot.

```sh
# Inspect arguments; choose a valid slot from the instrument's slot table.
python python/examples/runtime_bootloader_handoff.py --help
# Example only: use slot 1 only if that is the intended valid runtime image.
python python/examples/runtime_bootloader_handoff.py --home . --boot-slot 1 --boot-back
```

The workflow acquires runtime readiness, requests service, refreshes status and optionally boots back. Native owns quiescence and readiness on the same handle. Do not close/re-enumerate a new handle, clear an error and continue, or treat an UNKNOWN mode as success.

Omitting `--boot-back` intentionally leaves the device in service. After boot-back verify the expected runtime identity and authorized profile. This checks a lifecycle transition; it is not by itself an acquisition or image-integrity qualification.

[Tutorials](index.md) · [Runtime handoff](../02_hardware/2_12_runtime_handoff.md)
