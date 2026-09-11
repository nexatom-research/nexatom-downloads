## DLL Load and Version Check

The fundamental validation of the SDK installation is verifying that the native C library (`nexatomTT.dll`) can be discovered, loaded into memory, and executed by the Python wrapper.

**Relevant scripts:**
*   `load_smoke.py`
*   `version_info.py`

### Workflow

The version check scripts execute a minimal initialization sequence without opening USB connections or dispatching hardware commands.

1.  **Library Initialization:** The script instantiates the `NexatomLibrary` class. The constructor automatically searches the `NEXATOMTT_HOME` environment variable, the current working directory, and the module directory to resolve the path to the native `.dll`.
2.  **Version Queries:** The script invokes `lib.version()` and `lib.library_info()`. These methods map directly to the native `nexatom_tt_get_version()` and `nexatom_tt_get_library_info()` C API functions.
3.  **Validation:** If the script prints the version string without raising an `OSError` or `NexatomError`, the runtime dependencies are confirmed. This ensures that the MinGW standard libraries and FTDI D3XX driver are correctly co-located and resolvable by the Windows loader.

### Execution

To run the version check, execute the script from the command line:

```powershell
python python\examples\version_info.py
```

**Expected Output:**

```text
1.0.0
NexatomTT Library v1.0.0 - Built with C++17
```

To run the broader smoke test, which includes command-line argument parsing for explicit SDK paths:

```powershell
python python\examples\load_smoke.py --home .
```

**Expected Output:**

```text
Loaded NexatomTT 1.0.0
Python bindings are available for discovery, lifecycle, and realtime callbacks.
```