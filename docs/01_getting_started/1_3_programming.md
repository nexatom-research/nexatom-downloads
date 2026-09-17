# 1.3 Programming languages

## C / C++ integration

Include the supplied `include/nexatomtt_c_api.h`. It is the public C ABI and supports C++ consumers; do not copy internal headers or redefine the export macro. The extracted SDK's `examples/sdk/` project contains processed and raw C/C++ templates, shared validation helpers and its own build instructions.

Windows examples use the documented MinGW toolchain/import library. Linux examples link the matching shared library. Keep runtime dependencies reachable as described by that project's README. Do not assume an arbitrary compiler or a DLL from another SDK is interchangeable.

Each example checks errors, acquires runtime readiness, reads the profile, validates settings, prepares callbacks/files while quiet, starts measurement, validates results and completes cleanup. Adapt these stages instead of starting from a bare `connect()` call.

## Python integration

For a custom program launched outside the packaged examples, put the package's `python/` directory on the Python import path. Then:

```python
from nexatomtt import NexatomLibrary, RuntimeBootOptions, open_runtime_device

library = NexatomLibrary(home="/path/to/extracted-sdk")
devices = library.discover_devices()
if len(devices) != 1:
    # Require an intentional selection instead of silently choosing another unit.
    raise RuntimeError("Connect one instrument, or select an enumerated full serial")
with open_runtime_device(library, devices[0],
                         options=RuntimeBootOptions(timeout_ms=20000)) as device:
    # Native readiness establishes authority; it does not start our measurement.
    profile = device.get_device_profile()
    print(hex(profile.product_model_id), hex(profile.effective_public_tdc_mask))
    # Add the complete processed or raw template here, including its cleanup.
```

The binding raises `NexatomError` for native API failures and exposes typed callbacks and controls for acquisition, files, logging, calibration, configuration views and supported advanced features. See the [Python reference](../05_api_reference/index.md).

## FFI / Foreign language bindings

Use the exact C declarations and natural platform alignment, including nested records and padding. `bool`, enum, pointer and by-value callback layouts must match; an on-disk format is not an FFI structure. The shipped Python binding is a concrete example, not permission to retain a borrowed C callback pointer.

Copy callback data before handing it to asynchronous work. Clearing an ordinary data callback is not a fence for an invocation already selected; retain its function and user data until native device destruction completes. Logging has a separate explicit quiescent unregister. Preserve cleanup errors; see [callback lifetime](../06_in_depth_guides/6_5_callback_thread_safety_and_data_lifetime.md).

[Getting started](index.md) · [C API](../07_c_api/index.md)
