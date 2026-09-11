# Software Overview

## [Architecture and Data Flow](index.md#architecture-and-data-flow)

The NexatomTT SDK architecture is designed to robustly manage high-bandwidth streaming data while exposing a deterministic, state-safe control surface to the host application.

### [Native library](index.md#native-library)

The core engine is implemented in modern C++ and compiled into a monolithic dynamic link library (`nexatomTT.dll`). It exclusively exposes a stable, flat `extern "C"` Application Binary Interface (ABI). This guarantees high-performance interoperation with Foreign Function Interfaces (FFI)—such as Python's `ctypes`—avoiding C++ name mangling, ABI mismatches, and standard library linkage complications.

### [Data pipeline](index.md#data-pipeline)

Data acquisition relies on a pipelined threading architecture to prevent hardware buffer overflows during host OS scheduling interruptions:

1.  **FTDI Transport:** A dedicated read thread fetches raw bulk transfers from the FTDI D3XX kernel driver, rapidly moving data into large, pre-allocated internal circular buffers.
2.  **Decoder:** An intermediate processing stage parses the proprietary binary framing protocol, reconstructing hardware events, timestamps, and diagnostic telemetry.
3.  **ProcessingThread:** A routing thread aggregates the decoded data structures into their respective software modules (e.g., TIHI, MFCO, CPS).
4.  **Callbacks:** Finally, completed data payloads are dispatched asynchronously to the host application via registered callback functions.

### [Thread safety model](index.md#thread-safety-model)

The native SDK library is inherently thread-safe. All configuration, system control, and lifecycle API functions are internally guarded by mutexes. It is safe to invoke `nexatom_tt_` functions from multiple host threads concurrently.

Conversely, host software must respect the callback dispatch model:
*   Data callbacks are invoked asynchronously by the SDK's internal background `ProcessingThread`.
*   If a host language callback (e.g., in Python) must update a GUI, modify shared state, or interact with an event loop, **the user is strictly responsible** for implementing appropriate thread-safety mechanisms (e.g., locks or thread-safe message queues) to bridge the background worker thread to the main execution thread.

### [Memory management](index.md#memory-management)

The native library completely encapsulates all internal memory allocations and hardware buffer lifecycles. Users are never required to manually allocate or free memory for device communication.

To maximize safety across language FFI boundaries, the SDK strictly employs **pass-by-value** semantics for data callbacks. When a callback fires (e.g., yielding a `nexatom_cps_data_t` or `nexatom_tihi_data_t` struct), the entire payload is copied into the host language's memory space.

## [Precompiled Libraries and Language Bindings](index.md#precompiled-libraries-and-language-bindings)

### [Native DLL and runtime dependencies](index.md#native-dll-and-runtime-dependencies)

The SDK distributes several precompiled binaries for the Windows x64 platform. Due to Windows library loading rules, **all of the following DLLs must be co-located in the same directory**:

*   `nexatomTT.dll` (~8.6 MB): The core NexatomTT library.
*   `FTD3XXWU.dll`: The proprietary FTDI D3XX runtime driver.
*   `libgcc_s_seh-1.dll`, `libstdc++-6.dll`, `libwinpthread-1.dll`: MinGW compiler runtimes.

### [Python package (`nexatomtt`)](index.md#python-package-nexatomtt)

The official Python wrapper provides object-oriented abstractions over the C API without introducing external dependencies. It relies purely on the Python standard library (`ctypes`).

*   `nexatomtt._native`: Raw, low-level `ctypes` bindings mappings.
*   `nexatomtt.analysis`: Helpers for Multi-Fold Coincidence (MFCO) pattern decoding.
*   `nexatomtt.runtime_boot`: The orchestrator for safe bootloader-to-runtime transitions (`open_runtime_device()`).

## [C API](index.md#c-api)

### [Header organization](index.md#header-organization)

The entire C API is defined in a single, comprehensive header file (`nexatomtt_c_api.h`). It is logically partitioned into constants, structs, callback typedefs, and functional API groups (e.g., Device Lifecycle, Hardware Config, Telemetry).

### [Error handling convention](index.md#error-handling-convention)

To ensure deterministic error checking across all languages, the SDK avoids C++ exceptions at the ABI boundary.

*   **Return Type:** All functions return a `nexatom_error_code_t` (`int32`).
*   **Success:** A return value of `0` (`NEXATOM_SUCCESS`) indicates success.
*   **Failure:** A negative value indicates an error (e.g., `NEXATOM_ERROR_DEVICE_NOT_FOUND`).
*   **Error Details:** The caller can invoke `nexatom_tt_get_last_error_message()` on the calling thread to retrieve a descriptive string of the most recent failure, or use `nexatom_tt_get_error_message(code)` for a generic lookup.

### [Opaque handle pattern](index.md#opaque-handle-pattern)

Device instances are managed using opaque pointers to prevent the host application from tampering with internal state structures:
*   `nexatom_tt_handle`: Represents an active session with a physical device. Must be explicitly freed via `nexatom_tt_destroy()`.
*   `nexatom_tt_time_tag_reader_t*`: Represents an offline binary file parser. Must be closed via `nexatom_tt_close_time_tag_reader()`.
