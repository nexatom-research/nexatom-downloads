# 6.6 Native runtime startup

Use native `connect_runtime` for measurement readiness. It handles a device already in runtime or one that needs to boot an existing valid image from firmware service. It retains the same native handle and waits for runtime/profile authorization within the supplied budget.

```python
from nexatomtt import NexatomLibrary, RuntimeBootOptions, open_runtime_device

library = NexatomLibrary(home="/path/to/sdk")
devices = library.discover_devices()
if len(devices) != 1:
    raise RuntimeError("Select one intended device before starting")
with open_runtime_device(library, devices[0],
                         options=RuntimeBootOptions(timeout_ms=20000)) as device:
    # Native has established readiness. Choose controls from this profile.
    profile = device.get_device_profile()
    print(hex(profile.product_model_id))
    # Connecting does not start the application's intended measurement.
```

The helper signature is `open_runtime_device(library, device_info, *, options=..., log=..., pre_connect_profile=...)`. The optional pre-connect hook is a controlled compatibility/inventory facility; normal discovery does not need a manual profile.

An explicit `preferred_slot` chooses the service-mode boot target. If runtime is already present, the helper completes native readiness there. It does not force a new boot, write an image or change the default slot. For deliberate service/boot round trips use [the existing tutorial](../03_tutorials/3_6_runtime_bootloader_handoff_validation.md).

Handle timeout, unsupported identity and absent valid images directly. Do not implement model-specific register sequences, re-enumeration loops or fixed-delay workarounds in the client. Service-entry/boot reset boundaries are owned by native; caller-selected measurement start remains explicit.

[In-depth guides](index.md)
