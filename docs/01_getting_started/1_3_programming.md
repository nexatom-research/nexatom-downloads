## Programming Languages

The NexatomTT SDK provides two first-class programming interfaces — a C ABI exported from `nexatomTT.dll` and a Python `ctypes` wrapper — plus explicit FFI design conventions that enable binding from any language with a C foreign-function interface.

### [C / C++ integration](1_3_programming.md#c-cpp-integration)

#### Header

The entire public API is declared in a single header file:

```
include/nexatomtt_c_api.h    (3 036 lines, 118 KB)
```

The header includes only C standard library headers (`<stdint.h>`, `<stdbool.h>`, `<stddef.h>`) and is wrapped in `extern "C"` guards for direct use from C++.

```c
#include "nexatomtt_c_api.h"
```

The header references `../core/export.hpp` for the `NEXATOMTT_API` export macro. This build-time header is not shipped in the SDK because the DLL is precompiled. The macro resolves to `__declspec(dllimport)` when consuming the SDK. For FFI consumers, it can be defined as empty:

```c
#define NEXATOMTT_API
#include "nexatomtt_c_api.h"
```

#### Linking

The SDK ships a precompiled `nexatomTT.dll`. Link against it at load time or use runtime dynamic loading.

**MSVC (implicit linking):**

```
cl /I include my_app.c nexatomTT.lib /Fe:my_app.exe
```

Generate the import library from the DLL if one is not provided:

```
dumpbin /exports nexatomTT.dll > exports.txt
# Create a .def file from exports, then:
lib /def:nexatomTT.def /out:nexatomTT.lib /machine:x64
```

**MinGW:**

```
gcc -I include -o my_app.exe my_app.c -L . -lnexatomTT
```

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

All API functions (except version/info queries) return `nexatom_error_code_t`:

```c
nexatom_error_code_t rc = nexatom_tt_connect(device, 5000);
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
| -99 | `NEXATOM_ERROR_INTERNAL` | Internal library error |

`nexatom_tt_get_error_message(code)` returns a static string for any error code. `nexatom_tt_get_last_error_message()` returns a detailed, context-specific message from the most recent failure (process-global, thread-local lifetime).

#### Minimal C example

```c
#include <stdio.h>
#include <stdbool.h>
#include "nexatomtt_c_api.h"

void on_cps(nexatom_cps_data_t data, void* ctx) {
    printf("CPS total=%u period=%u ms\n", data.total_count, data.measurement_period_ms);
}

int main(void) {
    printf("NexatomTT %s\n", nexatom_tt_get_version());

    nexatom_tt_info_t devices[8];
    size_t count = 0;
    nexatom_tt_discover_devices(devices, 8, &count);
    if (count == 0) return 1;

    nexatom_tt_handle device;
    nexatom_tt_create(&devices[0], &device);
    nexatom_tt_connect(device, 5000);

    nexatom_tt_set_count_rate_callback(device, on_cps, NULL);
    nexatom_tt_enable_system(device, true);
    nexatom_tt_set_output_type(device, NEXATOM_OUTPUT_REALTIME_DATA);

    /* Collect CPS callbacks for 5 seconds */
    Sleep(5000);

    nexatom_tt_set_output_type(device, NEXATOM_OUTPUT_NO_OUTPUT);
    nexatom_tt_enable_system(device, false);
    nexatom_tt_disconnect(device);
    nexatom_tt_destroy(device);
    return 0;
}
```

---

### [Python integration](1_3_programming.md#python-integration)

The `nexatomtt` Python package wraps the C ABI via `ctypes`. No compiled extensions, no `pip install` step, and no dependencies outside the Python standard library are required.

#### Package architecture

```
python/nexatomtt/
├── __init__.py        Package namespace — re-exports all public symbols
├── _native.py         ctypes bindings, structs, callbacks, wrapper classes
├── analysis.py        MFCO pattern histogram analysis helpers
└── runtime_boot.py    Bootloader-first device startup orchestrator
```

| Module | Role | Dependencies |
|---|---|---|
| `_native` | DLL loading, ctypes struct definitions, `NexatomLibrary` / `NexatomDevice` / `NexatomTimeTagReader` classes, callback CFUNCTYPE definitions | `ctypes`, `os`, `pathlib`, `csv`, `time` |
| `analysis` | Pure-Python MFCO pattern analysis (no native calls) | None (stdlib only) |
| `runtime_boot` | High-level boot orchestrator using `NexatomDevice` methods | `time`, `dataclasses` |
| `__init__` | Re-exports 87 symbols via `__all__` | Imports from above modules |

#### Wrapper classes

**`NexatomLibrary`** — DLL loader and factory.

```python
from nexatomtt import NexatomLibrary

lib = NexatomLibrary(home=".")     # Load nexatomTT.dll
print(lib.version())               # "0.1.0-preview.6"
print(lib.library_info())          # Detailed build metadata

devices = lib.discover_devices()   # → list[NexatomDeviceInfo]
device = lib.create_device(devices[0])  # → NexatomDevice
```

**`NexatomDevice`** — Device session handle (context manager).

```python
with lib.create_device(devices[0]) as device:
    device.connect(timeout_ms=5000)
    device.enable_system(True)
    device.set_output_type(NEXATOM_OUTPUT_REALTIME_DATA)
    # ... acquire ...
    device.enable_system(False)
# disconnect + destroy called automatically
```

Key method groups:

| Group | Methods |
|---|---|
| Connection | `connect()`, `disconnect()`, `destroy()`, `close()`, `is_connected()`, `state()`, `hardware_protocol_mode()` |
| System | `enable_system()`, `reset_peripherals()`, `set_output_type()`, `get_output_type()`, `set_cps_period_selector()`, `request_global_stop_all_modes()` |
| Channels | `set_channel_threshold()`, `set_channel_input_delay()`, `enable_channel_test_pulse()`, `set_channel_test_pulse_params()` |
| TIHI | `enable_time_histogram()`, `set_time_histogram_channels()`, `set_time_histogram_bin_width()`, `set_time_histogram_num_bins()`, `set_time_histogram_stop_conditions()`, `set_time_histogram_aggregation_mode()`, `start_time_histogram()`, `stop_time_histogram()` |
| MFCO | `enable_multifold_coincidence()`, `set_multifold_coincidence_channels()`, `set_multifold_coincidence_window()`, `set_multifold_coincidence_stop_conditions()`, `set_multifold_coincidence_aggregation_mode()`, `start_multifold_coincidence()`, `stop_multifold_coincidence()` |
| Telemetry | `enable_telemetry()`, `set_telemetry_mode()`, `request_telemetry()`, `get_telemetry()` |
| File saving | `set_time_tag_file_config()`, `enable_time_tag_file_saving()`, `disable_time_tag_file_saving()` |
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

Python callbacks passed to the native library are wrapped in `ctypes.CFUNCTYPE` objects. The `NexatomDevice` class retains references to these objects in an internal `_callbacks` dictionary to prevent garbage collection while the callback is registered. Replaced callbacks are moved to a `_retired_callbacks` list and released on the next registration cycle.

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

High-level orchestrator for bootloader-first device startup. Key exports:

| Symbol | Type | Description |
|---|---|---|
| `open_runtime_device()` | Context manager | Discover → boot → reconnect → yield connected runtime device |
| `RuntimeBootOptions` | Dataclass | `timeout_ms`, `mode_timeout_sec`, `poll_sec`, `max_devices`, `preferred_slot` |
| `RuntimeBootError` | Exception | Raised when device cannot reach runtime firmware |
| `select_runtime_slot()` | Function | Select best VALID slot: preferred → default → lowest valid |
| `protocol_mode_name()` | Function | `int` → `"RUNTIME"` / `"BOOTLOADER"` / `"UNKNOWN"` |
| `slot_state_name()` | Function | `int` → `"EMPTY"` / `"VALID"` / `"PENDING"` / `"CORRUPT"` |

#### Public API inventory (`__all__`)

The `nexatomtt` package exports 87 symbols via `__all__`:

| Category | Count | Examples |
|---|---|---|
| Integer constants | 55 | `NEXATOM_STATE_*`, `NEXATOM_OUTPUT_*`, `NEXATOM_AGGREGATION_*`, `NEXATOM_FIELD_UPDATE_PHASE_*`, `NEXATOM_MFCO_*`, `NEXATOM_TIHI_*`, `NEXATOM_TT_FILE_*` |
| ctypes structures | 20 | `NexatomDeviceInfo`, `NexatomCpsData`, `NexatomTihiData`, `NexatomMfcoData`, `NexatomTelemetryData`, `NexatomTimeTag`, `NexatomFieldUpdateProgress` |
| Wrapper classes | 3 | `NexatomLibrary`, `NexatomDevice`, `NexatomTimeTagReader` |
| Exception classes | 2 | `NexatomError`, `RuntimeBootError` |
| Dataclasses | 1 | `RuntimeBootOptions` |
| Functions | 6 | `find_default_home`, `open_runtime_device`, `pattern_channels`, `pattern_mask`, `top_patterns`, `select_runtime_slot` |

---

### [FFI / Foreign language bindings](1_3_programming.md#ffi-foreign-language-bindings)

The C ABI is designed for binding from any language with a C FFI capability (Dart, C#, Rust, Go, etc.). Three design conventions simplify cross-language integration.

#### Explicit struct padding

All structs contain explicit `_padding` fields that make compiler-inserted alignment padding visible to FFI code generators. This ensures identical memory layout across compilers, platforms, and language runtimes.

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

All data callbacks pass their payload struct **by value**, not by pointer:

```c
typedef void (*nexatom_count_rate_callback)(
    nexatom_cps_data_t data,     /* ← by value, not by pointer */
    void* user_data
);
```

This design has three consequences for FFI consumers:

1. **No lifetime management.** The callback data is a stack copy owned by the callee. There are no dangling pointers, no reference counting, and no free-after-use bugs. The data remains valid indefinitely after the callback returns.

2. **Thread safety.** The data can be safely dispatched to a UI thread, event loop, or async queue without synchronizing with the native library's internal buffers.

3. **FFI function signature.** The callback's first parameter is the full struct, not a pointer. FFI frameworks must declare the callback type accordingly:

```python
# Python ctypes
_CountRateCallback = ctypes.CFUNCTYPE(
    None,              # return void
    NexatomCpsData,    # data — by value
    ctypes.c_void_p,   # user_data
)
```

The following 7 data callbacks use pass-by-value:

| Callback | Payload struct | Approximate size |
|---|---|---|
| `nexatom_time_histogram_callback` | `nexatom_tihi_callback_data_t` | ~8.3 KB |
| `nexatom_multifold_coincidence_callback` | `nexatom_mfco_callback_data_t` | ~1.2 KB |
| `nexatom_count_rate_callback` | `nexatom_cps_data_t` | 44 B |
| `nexatom_multi_tau_correlation_callback` | `nexatom_corm_callback_data_t` | ~2.4 KB |
| `nexatom_linear_correlation_callback` | `nexatom_corl_callback_data_t` | ~1.4 KB |
| `nexatom_telemetry_callback` | `nexatom_telemetry_data_t` | ~64 B |
| `nexatom_config_dump_callback` | `nexatom_config_dump_data_t` | ~1 KB |

The `nexatom_log_callback` also passes `nexatom_log_record_t` by value (~1.1 KB). The `nexatom_connection_status_callback` passes two scalar `int` parameters (no struct).

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
2. **Define callback types** — Declare function pointer types with the payload struct passed by value (not by pointer). Register callbacks before enabling output.
3. **Map handles to opaque pointers** — Use `void*` or the language's equivalent. Pair every create with a destroy in a finalizer or destructor.
4. **Map error codes** — Check every `nexatom_error_code_t` return value. Call `nexatom_tt_get_last_error_message()` for diagnostic text on failure.
5. **Locate the DLL** — Load `nexatomTT.dll` by absolute path and register its directory for dependent DLL resolution (`os.add_dll_directory` on Windows, `SetDllDirectoryA`, or equivalent).
