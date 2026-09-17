## Programming Languages

The NexatomTT SDK provides C and C++ applications with a stable C ABI and Python applications with a `ctypes` wrapper over the same API. Windows loads `nexatomTT.dll`; Linux loads `libnexatomTT.so`. The package includes commented C, C++ and Python measurement examples. Other languages can bind the C interface using the conventions explained below.

### [C / C++ integration](1_3_programming.md#c-cpp-integration)

#### Header

The entire public API is declared in a single header file:

```
include/nexatomtt_c_api.h
```

The header includes only C standard library headers (`<stdint.h>`, `<stdbool.h>`, `<stddef.h>`) and is wrapped in `extern "C"` guards for direct use from C++.

```c
#include "nexatomtt_c_api.h"
```

The packaged header is standalone: it supplies the public export declarations without private implementation headers. Include it directly, without defining away `NEXATOMTT_API`. C++ consumers use the same ABI; the package's C++ example adds resource ownership with `std::unique_ptr`.

#### Linking

The SDK ships a precompiled library. Link with the supplied platform library or use runtime dynamic loading. The simplest complete C/C++ starting point is the standalone `examples/sdk` project:

```sh
cmake -S examples/sdk -B build-examples -DNEXATOMTT_SDK_ROOT=/absolute/path/to/sdk
cmake --build build-examples
```

Use an absolute Windows path with the MSYS2 UCRT64 toolchain on Windows; use GCC or Clang on Linux. This project builds `hardware_c` and `hardware_cpp`. It copies Windows runtime DLLs beside the executables and sets the Linux runtime search path to the selected SDK.

**Windows compiler choice:**

The supplied Windows example project supports MinGW and includes `lib/nexatomTT.dll.a`. It does not ship an MSVC `.lib` or an MSVC example build. An MSVC integration needs a compatible import library and verification of the callback/structure ABI; do not substitute private C++ headers or mix compiler runtimes.

**MinGW:**

```
gcc -I include -o my_app.exe my_app.c lib/nexatomTT.dll.a
```

**Linux (from the SDK root):**

```sh
gcc -I include -o my_app my_app.c -L . -lnexatomTT -Wl,-rpath,'$ORIGIN'
```

This direct-link example places `my_app` beside the shared library. Keep the extracted SDK libraries there; a relocated application needs a corresponding runtime search path.

**Runtime loading (Windows `LoadLibrary`):**

```c
HMODULE dll = LoadLibraryA("nexatomTT.dll");
typedef const char* (*GetVersionFn)(void);
GetVersionFn get_version = (GetVersionFn)GetProcAddress(dll, "nexatom_tt_get_version");
printf("Version: %s\n", get_version());
```

#### DLL co-location

All five DLL files must reside in the same directory as the executable, or the SDK root must be on the `PATH` environment variable:

| DLL | Purpose |
|---|---|
| `nexatomTT.dll` | Core library |
| `FTD3XXWU.dll` | FTDI D3XX USB transport |
| `libgcc_s_seh-1.dll` | GCC exception handling |
| `libstdc++-6.dll` | C++ standard library |
| `libwinpthread-1.dll` | POSIX threads |

#### Error handling convention

Most control functions return `nexatom_error_code_t`. Always follow the declared return type: version functions return strings, some functions return scalar values, and destruction returns `void`.

```c
nexatom_error_code_t rc = nexatom_tt_connect_runtime(device, 20000);
if (rc != NEXATOM_SUCCESS) {
    const char* detail = nexatom_tt_get_last_error_message();
    fprintf(stderr, "Connect failed (%d): %s\n", rc, detail);
}
```

| Code | Constant | Meaning |
|---|---|---|
| 0 | `NEXATOM_SUCCESS` | Operation completed |
| -1 | `NEXATOM_ERROR_INVALID_PARAMETER` | Invalid argument |
| -2 | `NEXATOM_ERROR_NOT_CONNECTED` | Device not connected |
| -4 | `NEXATOM_ERROR_CONNECTION_FAILED` | USB connection failed |
| -5 | `NEXATOM_ERROR_TIMEOUT` | Operation timed out |
| -7 | `NEXATOM_ERROR_ACQUISITION_RUNNING` | Stop acquisition first |
| -11 | `NEXATOM_ERROR_FILE_IO` | File I/O failure |
| -12 | `NEXATOM_ERROR_NOT_SUPPORTED` | Operation unsupported by the active device contract |
| -99 | `NEXATOM_ERROR_INTERNAL` | Internal library error |

`nexatom_tt_get_error_message(code)` returns a static description. `nexatom_tt_get_last_error_message()` returns more detailed diagnostic text. Copy that text promptly into application-owned storage if it must outlive subsequent API calls; pair it with the return code from the operation that failed.

#### Minimal C example

This small C program demonstrates discovery, native runtime entry, callbacks and checked cleanup. It uses an external signal; zero CPS values can be legitimate when no events arrive. For channel configuration, test pulses, saving and result checks use the complete `examples/sdk/hardware.c` project.

```c
#ifndef _WIN32
#define _POSIX_C_SOURCE 200809L
#endif
#include <stdio.h>
#include <stdbool.h>
#include <inttypes.h>
#ifdef _WIN32
#include <windows.h>
#else
#include <time.h>
#include <errno.h>
#endif
#include "nexatomtt_c_api.h"

static int check(nexatom_error_code_t rc, const char *operation) {
    if (rc == NEXATOM_SUCCESS) return 0;
    fprintf(stderr, "%s failed (%d): %s\n", operation, (int)rc,
            nexatom_tt_get_last_error_message());
    return 1;
}

static void on_cps(nexatom_cps_data_t data, void *context) {
    (void)context;
    /* Historical field name: total_count is already a rate in Hz. */
    printf("CPS total=%" PRIu32 " Hz, window=%" PRIu32 " ms\n",
           data.total_count, data.measurement_period_ms);
}

int main(void) {
    nexatom_tt_info_t devices[8];
    size_t count = 0;
    nexatom_tt_handle device = NULL;
    bool connected = false;
    int failed = 0;
#define CALL(operation) do { \
    if (check((operation), #operation)) { failed = 1; goto cleanup; } \
} while (0)

    printf("NexatomTT %s\n", nexatom_tt_get_version());
    CALL(nexatom_tt_discover_devices(devices, 8, &count));
    if (count != 1) {
        fprintf(stderr, "Connect one intended instrument for this example.\n");
        return 1;
    }
    CALL(nexatom_tt_create(&devices[0], &device));
    CALL(nexatom_tt_connect_runtime(device, 20000));
    connected = true;
    /* Native checks whether each requested operation is supported. */
    CALL(nexatom_tt_set_output_type(device, NEXATOM_OUTPUT_NO_OUTPUT));
    CALL(nexatom_tt_enable_system(device, false));
    CALL(nexatom_tt_set_count_rate_callback(device, on_cps, NULL));
    CALL(nexatom_tt_set_cps_period_selector(device, 0));
    CALL(nexatom_tt_set_output_type(device, NEXATOM_OUTPUT_REALTIME_DATA));
    CALL(nexatom_tt_enable_system(device, true));

    /* This is the observation interval, not a startup/readiness delay. */
#ifdef _WIN32
    Sleep(5000);
#else
    struct timespec remaining = {5, 0};
    while (nanosleep(&remaining, &remaining) != 0 && errno == EINTR) {}
#endif

cleanup:
    if (connected) {
        failed |= check(nexatom_tt_set_output_type(device, NEXATOM_OUTPUT_NO_OUTPUT), "stop output");
        failed |= check(nexatom_tt_enable_system(device, false), "disable system");
        failed |= check(nexatom_tt_disconnect(device), "disconnect");
    }
    /* Keep callback state alive until this point, even after an error. */
    if (device) nexatom_tt_destroy(device);
    return failed ? 1 : 0;
#undef CALL
}
```

<a id="c-cpp-examples"></a>

#### C++ ownership and the complete examples

The C++ template uses `std::unique_ptr` with a custom deleter to pair handle creation with cleanup. Its `hardware.cpp` calls the same `acquisition.c` measurement code used by `hardware.c`, so threshold, edge, delay, hysteresis, saving and analysis follow the same sequence.

```sh
./build-examples/hardware_c processed --channels 0,1 --threshold-mv 500 --edge rising --delay-ps 0 --hysteresis-mv 10 --internal-test --duration-sec 5 --output-dir processed-c
./build-examples/hardware_cpp raw --channels 0,1 --internal-test --duration-sec 5 --output-dir raw-cpp --decode-csv
```

On Windows use `build-examples\hardware_c.exe` and `hardware_cpp.exe`. Each command needs a new output directory whose parent exists. `processed` records CPS/TIHI/MFCO CSV and `summary.txt`; `raw` records NXTT, optional `time_tags.csv` and `summary.txt`. Both executables accept either mode. Edit `configure()` for per-channel values and `start_processed()` for timing windows. Run commands separately with one active instrument.

---

### [Python integration](1_3_programming.md#python-integration)

The `nexatomtt` Python package wraps the C ABI via `ctypes`. No compiled extensions, no `pip install` step, and no dependencies outside the Python standard library are required.

#### Package architecture

```
python/nexatomtt/
├── __init__.py        Package namespace — re-exports all public symbols
├── _native.py         ctypes bindings, structs, callbacks, wrapper classes
├── analysis.py        MFCO pattern histogram analysis helpers
└── runtime_boot.py    Context manager around native runtime entry
```

| Module | Role | Dependencies |
|---|---|---|
| `_native` | DLL loading, ctypes struct definitions, `NexatomLibrary` / `NexatomDevice` / `NexatomTimeTagReader` classes, callback CFUNCTYPE definitions | `ctypes`, `os`, `pathlib`, `csv`, `time` |
| `analysis` | Pure-Python MFCO pattern analysis (no native calls) | None (stdlib only) |
| `runtime_boot` | Native runtime entry and explicit existing-slot service helpers | `time`, `dataclasses` |
| `__init__` | Public re-exports, including expanded device controls and records | Imports from the package modules |

Additional modules group newer controls and structures. Import public names from `nexatomtt`; applications do not need to depend on the internal module split. Place your script in the SDK's `python/` directory or add that directory to `PYTHONPATH` before importing.

#### Wrapper classes

**`NexatomLibrary`** — DLL loader and factory.

```python
from nexatomtt import NexatomLibrary

lib = NexatomLibrary(home=".")     # Load nexatomTT.dll
print(lib.version())               # Native library version; archive version is in manifest.json
print(lib.library_info())          # Detailed build metadata

devices = lib.discover_devices()   # → list[NexatomDeviceInfo]
device = lib.create_device(devices[0])  # → NexatomDevice
```

**`NexatomDevice`** — Device session handle (context manager).

```python
from nexatomtt import open_runtime_device, RuntimeBootOptions

with open_runtime_device(lib, devices[0], RuntimeBootOptions(timeout_ms=20000)) as device:
    profile = device.get_device_profile()
    print(f"Usable inputs: 0x{profile.effective_public_tdc_mask:x}")
    # Configure and acquire here, following the complete example below.
    device.disconnect()  # Explicit call lets the application observe failure.
# The context destroys the handle, including if a preceding operation fails.
```

Key method groups:

| Group | Methods |
|---|---|
| Connection | `connect_runtime()`, `get_device_profile()`, `get_capabilities()`, `connect()`, `disconnect()`, `destroy()`, `close()`, `is_connected()`, `state()`, `hardware_protocol_mode()` |
| System | `enable_system()`, `reset_peripherals()`, `set_output_type()`, `get_output_type()`, `set_cps_period_selector()`, `request_global_stop_all_modes()` |
| Channels | `set_channel_threshold()`, `set_channel_edge_type()`, `set_channel_input_delay()`, `set_channel_hysteresis()`, `enable_channel_test_pulse()`, `set_channel_test_pulse_params()` |
| TIHI | `enable_time_histogram()`, `set_time_histogram_channels()`, `set_time_histogram_bin_width()`, `set_time_histogram_num_bins()`, `set_time_histogram_stop_conditions()`, `set_time_histogram_aggregation_mode()`, `start_time_histogram()`, `stop_time_histogram()` |
| MFCO | `enable_multifold_coincidence()`, `set_multifold_coincidence_channels()`, `set_multifold_coincidence_window()`, `set_multifold_coincidence_stop_conditions()`, `set_multifold_coincidence_aggregation_mode()`, `start_multifold_coincidence()`, `stop_multifold_coincidence()` |
| Telemetry | `enable_telemetry()`, `set_telemetry_mode()`, `request_telemetry()`, `get_telemetry()` |
| File saving | Raw `set_time_tag_file_config()`, `enable_time_tag_file_saving()`, `disable_time_tag_file_saving()`; processed `set_processed_file_config()`, `enable_processed_file_saving()`, `disable_processed_file_saving()` |
| Callbacks | `set_connection_status_callback()`, `set_count_rate_callback()`, `set_telemetry_callback()`, `set_time_histogram_callback()`, `set_multifold_coincidence_callback()`, `clear_callbacks()` |
| Field update | `request_field_upgrade_service_entry()`, `clear_field_upgrade_service_entry_request()`, `refresh_field_update_status()`, `load_field_update_image()`, `set_field_update_default_slot()`, `set_field_update_default_slot_with_status()`, `boot_field_update_slot()` |

**`NexatomTimeTagReader`** — Offline `.nxtt` binary file reader (context manager).

```python
with lib.open_time_tag_file_reader("capture.nxtt") as reader:
    header = reader.header()           # → NexatomTimeTagFileHeader
    for tag in reader.iter_tags():     # generator, batch_size=4096
        print(tag.timestamp_ps, tag.channel)

# Or bulk export:
with lib.open_time_tag_file_reader("capture.nxtt") as reader:
    rows = reader.export_csv("output.csv")  # → int (row count)
```

Series reader for multi-file captures:

```python
reader = lib.open_time_tag_series_reader(
    directory="captures/",
    series_key="nexatomtt_capture",
    seq_start=1,
    seq_end=0,    # 0 = all contiguous files
)
```

#### Error handling

All native API calls are checked via `NexatomLibrary.check(code)`. On non-zero return codes, `NexatomError` is raised:

```python
from nexatomtt import NexatomError

try:
    device.connect(timeout_ms=1000)
except NexatomError as e:
    print(e.code)            # -5 (NEXATOM_ERROR_TIMEOUT)
    print(e.native_message)  # Detailed failure text
```

#### Callback reference management

Python callbacks passed to the native library are wrapped in `ctypes.CFUNCTYPE` objects. The device retains current and retired callback wrappers through native handle destruction, because a worker may already have selected a callback when it is replaced or cleared. Clearing a callback is not a wait for every in-flight invocation to finish. Keep application callback state alive until device teardown completes.

```python
def on_cps(data):
    print(f"CPS total={data.total_count}")

device.set_count_rate_callback(on_cps)
# The ctypes wrapper around on_cps is kept alive in device._callbacks
```

#### Analysis helpers (`nexatomtt.analysis`)

Pure-Python utility functions for 8-channel, 256-bin MFCO pattern histograms:

| Function | Signature | Description |
|---|---|---|
| `pattern_channels` | `(pattern: int) → list[int]` | Decode 8-bit pattern to channel list. `pattern_channels(3)` → `[0, 1]` |
| `pattern_mask` | `(channels: Iterable[int]) → int` | Encode channel list to bitmask. `pattern_mask([0, 1])` → `3` |
| `exact_pattern_count` | `(bins, channels) → int` | Count for exactly the specified channel combination |
| `contains_channels_count` | `(bins, channels) → int` | Sum counts for all patterns containing all specified channels |
| `order_counts` | `(bins) → dict` | Group by order: `{"singles": n, "doubles": n, "triples": n, "higher": n}` |
| `top_patterns` | `(bins, limit=10) → list[dict]` | Top N non-zero patterns sorted by count descending |

#### Runtime boot module (`nexatomtt.runtime_boot`)

The default runtime helper delegates startup to native `connect_runtime` on one handle. Legacy Zynq attaches directly; bootloader-equipped instruments can boot an existing valid slot. Key exports:

| Symbol | Type | Description |
|---|---|---|
| `open_runtime_device()` | Context manager | Create one handle → native runtime entry → yield ready device → cleanup |
| `RuntimeBootOptions` | Dataclass | `timeout_ms`, `mode_timeout_sec`, `poll_sec`, `max_devices`, `preferred_slot` |
| `RuntimeBootError` | Exception | Raised when device cannot reach runtime firmware |
| `select_runtime_slot()` | Function | Select best VALID slot: preferred → default → lowest valid |
| `protocol_mode_name()` | Function | `int` → `"RUNTIME"` / `"BOOTLOADER"` / `"UNKNOWN"` |
| `slot_state_name()` | Function | `int` → `"EMPTY"` / `"VALID"` / `"PENDING"` / `"CORRUPT"` |

#### Public API inventory (`__all__`)

The public namespace includes the groups below. Inspect `nexatomtt.__all__` for the exact inventory in the installed package; new controls have expanded the earlier release's inventory.

| Category | Examples |
|---|---|
| Integer constants | `NEXATOM_STATE_*`, `NEXATOM_OUTPUT_*`, `NEXATOM_AGGREGATION_*`, `NEXATOM_TT_FEATURE_*`, file-format and completion constants |
| Structures | `NexatomDeviceInfo`, device profiles, capabilities, CPS/TIHI/MFCO/correlation records, telemetry, time tags and field-update progress |
| Wrapper classes | `NexatomLibrary`, `NexatomDevice`, `NexatomTimeTagReader` |
| Exceptions | `NexatomError`, `RuntimeBootError`, `NexatomDtcApplyRejected` |
| Runtime options | `RuntimeBootOptions` |
| Helpers | `find_default_home`, `open_runtime_device`, MFCO pattern helpers and slot selection |

---

### [FFI / Foreign language bindings](1_3_programming.md#ffi-foreign-language-bindings)

The C ABI is designed for binding from any language with a C FFI capability (Dart, C#, Rust, Go, etc.). Three design conventions simplify cross-language integration.

#### Explicit struct padding

Many fixed-layout structs contain explicit `_padding` fields that make alignment visible to FFI code generators. Mirror the exact header for your SDK and platform; pointer-containing or versioned structures also need their declared alignment, sizes and lifetimes. Padding fields alone do not prove an ABI match on an arbitrary platform.

**Convention:** A `_padding` array of `uint8_t` is inserted after narrow fields (`uint8_t`, `bool`) before wider fields (`uint32_t`, `uint64_t`, `double`) to align to the wider type's natural boundary.

**Example — `nexatom_cps_data_t`:**

```c
typedef struct nexatom_cps_data {
    uint32_t total_count;            /* offset  0, 4 bytes */
    uint32_t measurement_period_ms;  /* offset  4, 4 bytes */
    uint8_t  num_channels;           /* offset  8, 1 byte  */
    uint8_t  _padding[3];            /* offset  9, 3 bytes — explicit alignment */
    uint32_t counts[8];              /* offset 12, 32 bytes */
} nexatom_cps_data_t;               /* total:  44 bytes    */
```

The corresponding Dart FFI, Python ctypes, and C# struct definitions must include these padding fields at the same offsets. The C header comment block above each callback struct documents the exact byte-level memory layout.

**Python ctypes mirror:**

```python
class NexatomCpsData(ctypes.Structure):
    _fields_ = [
        ("total_count",           ctypes.c_uint32),
        ("measurement_period_ms", ctypes.c_uint32),
        ("num_channels",          ctypes.c_uint8),
        ("_padding",              ctypes.c_uint8 * 3),
        ("counts",                ctypes.c_uint32 * 8),
    ]
```

#### Pass-by-value callback semantics

The ordinary fixed-record callbacks pass their payload struct **by value**, for example:

```c
typedef void (*nexatom_count_rate_callback)(
    nexatom_cps_data_t data,     /* ← by value, not by pointer */
    void* user_data
);
```

This design has three consequences for FFI consumers:

1. **Copy before retaining.** In C/C++, the callback parameter lives for that invocation. Copy it into application-owned storage before returning if it is needed later; never retain its stack address. Fixed inline arrays are included in that copy. Pointer-bearing views require separate handling.

2. **Dispatch owned data.** Send the copied record to a synchronized application queue for slower processing or a UI update. The Python binding supplies owned copies for supported data callbacks. Thread safety of your own shared state remains your responsibility.

3. **FFI function signature.** The callback's first parameter is the full struct, not a pointer. FFI frameworks must declare the callback type accordingly:

```python
# Python ctypes
_CountRateCallback = ctypes.CFUNCTYPE(
    None,              # return void
    NexatomCpsData,    # data — by value
    ctypes.c_void_p,   # user_data
)
```

These legacy fixed-record callbacks use pass-by-value (sizes should be checked against the installed header):

| Callback | Payload struct | Approximate size |
|---|---|---|
| `nexatom_time_histogram_callback` | `nexatom_tihi_callback_data_t` | ~8.3 KB |
| `nexatom_multifold_coincidence_callback` | `nexatom_mfco_callback_data_t` | ~1.2 KB |
| `nexatom_count_rate_callback` | `nexatom_cps_data_t` | 44 B |
| `nexatom_multi_tau_correlation_callback` | `nexatom_corm_callback_data_t` | ~2.4 KB |
| `nexatom_linear_correlation_callback` | `nexatom_corl_callback_data_t` | ~1.4 KB |
| `nexatom_telemetry_callback` | `nexatom_telemetry_data_t` | ~64 B |
| `nexatom_config_dump_callback` | `nexatom_config_dump_data_t` | Fixed legacy summary; see the separate versioned view below |

The versioned configuration-dump callback is a borrowed pointer/view with a record array. A C application must copy the view and referenced records before callback return; copying only the pointer does not retain them. The Python view wrapper owns its record copy. See [callback data lifetime](../06_in_depth_guides/6_5_callback_thread_safety_and_data_lifetime.md) for both cases.

The `nexatom_log_callback` passes `nexatom_log_record_t` by value. The `nexatom_connection_status_callback` passes two scalar `int` parameters. Follow each callback's exact typedef rather than assuming all callback signatures are alike.

> **Performance note.** The largest by-value callback payload is `nexatom_tihi_callback_data_t` at ~8.3 KB (1024 histogram bins + fitting results + fitted curve). At typical callback rates of 1–10 Hz, the copy overhead is negligible compared to acquisition time.

#### Opaque handle pattern

Device sessions and file readers are managed through opaque handles — forward-declared struct pointers that hide the C++ implementation behind the C ABI:

```c
typedef struct nexatom_tt*                  nexatom_tt_handle;
typedef struct nexatom_tt_time_tag_reader   nexatom_tt_time_tag_reader_t;
```

FFI consumers should treat these as `void*` or equivalent opaque pointer types. The handles are created and destroyed through paired API functions:

| Create | Destroy |
|---|---|
| `nexatom_tt_create()` → `nexatom_tt_handle` | `nexatom_tt_destroy(handle)` |
| `nexatom_tt_open_time_tag_file_reader()` → `nexatom_tt_time_tag_reader_t*` | `nexatom_tt_close_time_tag_reader(reader)` |

#### Building a new language binding

To create bindings for a new language:

1. **Define structs** — Mirror every `typedef struct` from the header, preserving field order, types, and `_padding` arrays. Verify `sizeof` matches the C layout.
2. **Define callback types** — Match each typedef, including by-value records and borrowed pointer/view exceptions. Register callbacks before enabling output and keep callback objects/state alive until teardown completes.
3. **Map handles to opaque pointers** — Use `void*` or the language's equivalent. Pair every create with a destroy in a finalizer or destructor.
4. **Map error codes** — Check every `nexatom_error_code_t` return value. Call `nexatom_tt_get_last_error_message()` for diagnostic text on failure.
5. **Locate the library** — Load the matching platform library by absolute path. On Windows register its directory for dependency resolution; on Linux retain the package layout and runtime search path.
