## DLL Load and Version Check

The first installation check verifies that the native library (`nexatomTT.dll` on Windows or `libnexatomTT.so` on Linux) can be located, loaded and called by the Python wrapper. No connected instrument is needed.

**Relevant scripts:**
*   `load_smoke.py`
*   `version_info.py`

### Workflow

The version check scripts execute a minimal initialization sequence without opening USB connections or dispatching hardware commands.

1.  **Library Initialization:** The script instantiates `NexatomLibrary`. An explicit `home` takes precedence; otherwise the loader checks `NEXATOMTT_HOME` and candidate directories relative to the Python module. Run the supplied examples from the SDK root with `--home .` when that option is available.
2.  **Version Queries:** The script invokes `lib.version()` and `lib.library_info()`. These methods map directly to the native `nexatom_tt_get_version()` and `nexatom_tt_get_library_info()` C API functions.
3.  **Validation:** A successful load and version query show that the native library and its load-time dependencies resolve. The wrapper also verifies the adjacent package manifest. This does not test USB permissions, a Windows device-driver installation or acquisition; those are exercised when opening an instrument.

### Execution

To run the version check, execute the script from the command line:

```powershell
python python/examples/version_info.py
```

**Output interpretation:**

```text
<native library version>
<native library/build identification>
```

To run the broader smoke test, which includes command-line argument parsing for explicit SDK paths:

```powershell
python python/examples/load_smoke.py --home .
```

The load smoke prints this form:

```text
Loaded NexatomTT <native library version>
Python bindings are available for discovery, lifecycle, and realtime callbacks.
```

The native version string and SDK package version are distinct: read `manifest.json` for the archive version. Do not treat a native `1.0.0` string as evidence that the package is an older SDK. On Linux use `python3` if needed. After this succeeds, continue to [runtime entry](3_2_booting_into_runtime_from_bootloader.md).
