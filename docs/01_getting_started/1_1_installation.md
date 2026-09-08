# Installation Guide

### System requirements

| Requirement | Detail |
|---|---|
| Operating system | Windows 10 or later, 64-bit |
| CPU architecture | x86-64 (AMD64) |
| USB | USB 3.0 port (USB 2.0 functional at reduced throughput) |
| FTDI D3XX driver | Required for USB transport to the UTT810 |
| Python (optional) | CPython 3.10 or later (for Python wrapper and examples) |
| Disk space | ~16 MB (SDK package) |

No compiler, build system, or `pip install` step is required. The Python wrapper uses only the standard library (`ctypes`, `pathlib`, `os`, `csv`, `time`, `argparse`). The optional live plotting example (`tihi_mfco_matplotlib.py`) requires `matplotlib`.

---

### [FTDI D3XX driver installation](1_1_installation.md#ftdi-d3xx-driver-installation)

The UTT810 communicates with the host PC through an FTDI FT60x USB 3.0 controller. The host-side FTDI D3XX driver must be installed before the SDK can access the device.

**Step 1.** Download the D3XX driver package from the FTDI website:

```
https://ftdichip.com/drivers/d3xx-drivers/
```

Select the **Windows** driver package for your architecture (x64).

**Step 2.** Extract the archive and run the driver installer. Follow the FTDI installation prompts.

**Step 3.** Connect the UTT810 to a USB 3.0 port. Windows Device Manager should show the device under **Universal Serial Bus controllers** as an FTDI D3XX device.

> **Note.** The SDK ships a bundled `FTD3XXWU.dll` runtime library (515 KB) alongside `nexatomTT.dll`. This is the user-mode runtime that the native library links against at load time. It is **not** a substitute for the kernel-mode FTDI D3XX driver — both must be present. The bundled DLL is proprietary software from Future Technology Devices International Limited (FTDI Chip). See `THIRD_PARTY_NOTICES.md` for license terms.

---

### SDK package contents

Extract the SDK archive. The resulting directory contains the following files:

| File | Size | Role |
|---|---|---|
| `nexatomTT.dll` | 8.6 MB | Core native library — C ABI surface for all SDK functionality |
| `FTD3XXWU.dll` | 515 KB | FTDI D3XX user-mode runtime (USB 3.0 transport) |
| `libgcc_s_seh-1.dll` | 151 KB | MinGW runtime — GCC structured exception handling |
| `libstdc++-6.dll` | 2.5 MB | MinGW runtime — C++ standard library |
| `libwinpthread-1.dll` | 64 KB | MinGW runtime — POSIX threads implementation |
| `manifest.json` | 8.5 KB | SDK version, file SHA-256 hashes, build provenance |
| `LICENSE.txt` | 2.7 KB | SDK license terms |
| `THIRD_PARTY_NOTICES.md` | 2.7 KB | Third-party component licenses (FTDI, GCC, HDF5) |

| Directory | Contents |
|---|---|
| `include/` | `nexatomtt_c_api.h` — public C ABI header (3 036 lines, 118 KB) |
| `python/nexatomtt/` | Python wrapper package: `__init__.py`, `_native.py`, `analysis.py`, `runtime_boot.py` |
| `python/examples/` | 8 example scripts + 1 Windows launcher (`.cmd`) + `README.md` |
| `firmware/` | `BOOT_001.bin` (UTT810 runtime firmware, 4.2 MB) + `firmware_manifest.json` |
| `docs/` | Documentation files |

**DLL co-location rule.** All five DLL files must reside in the same directory. The Python wrapper loads `nexatomTT.dll` by absolute path from the SDK root and calls `os.add_dll_directory()` so that Windows can resolve the dependent MinGW and FTDI runtime DLLs at load time. Moving individual DLLs to a separate directory will cause `OSError` on import.

```
nexatomtt-sdk-windows-x64/
├── nexatomTT.dll            ← loaded by ctypes.CDLL()
├── FTD3XXWU.dll             ← resolved via os.add_dll_directory()
├── libgcc_s_seh-1.dll       ← resolved via os.add_dll_directory()
├── libstdc++-6.dll          ← resolved via os.add_dll_directory()
└── libwinpthread-1.dll      ← resolved via os.add_dll_directory()
```

For C/C++ consumers, ensure the SDK root is on the DLL search path (e.g., place the executable in the SDK root, or add the SDK root to the `PATH` environment variable).

---

### Verifying the installation

Two verification steps confirm the SDK is correctly installed: a DLL load smoke test and a version info query. Neither requires connected hardware.

#### DLL load smoke test

From a terminal opened at the SDK root:

```powershell
python python\examples\load_smoke.py --home .
```

Expected output:

```
Loaded NexatomTT 0.1.0-preview.6
Python bindings are available for discovery, lifecycle, and realtime callbacks.
```

This confirms that:
- `nexatomTT.dll` was found and loaded via `ctypes.CDLL()`
- All dependent DLLs (`FTD3XXWU.dll`, MinGW runtimes) resolved successfully
- The C function `nexatom_tt_get_version()` returned a valid version string
- The `NexatomLibrary` class initialized without error

If the DLL is not found, the script raises `FileNotFoundError` with the searched path. If a dependent DLL is missing, it raises `OSError` with a diagnostic message.

#### Version and build info query

```powershell
python python\examples\version_info.py
```

This script calls both `nexatom_tt_get_version()` and `nexatom_tt_get_library_info()`:

```python
from nexatomtt import NexatomLibrary

lib = NexatomLibrary()
print(lib.version())       # → "0.1.0-preview.6"
print(lib.library_info())  # → detailed build metadata string
```

**C equivalent:**

```c
#include "nexatomtt_c_api.h"

const char* version = nexatom_tt_get_version();

char info[4096];
nexatom_error_code_t rc = nexatom_tt_get_library_info(info, sizeof(info));
```

`nexatom_tt_get_version()` returns a pointer to an internal constant string (do not free). `nexatom_tt_get_library_info()` writes a null-terminated build metadata string into the caller-supplied buffer and returns `NEXATOM_SUCCESS` (0) on success.

---

### SDK home discovery

The Python wrapper must locate the directory containing `nexatomTT.dll` at initialization time. This directory is called the **SDK home**. The `NexatomLibrary` constructor resolves it through the following precedence chain:

**1. Explicit `home` parameter**

```python
lib = NexatomLibrary(home=r"C:\nexatomtt-sdk-windows-x64")
```

When `home` is supplied (via constructor argument or `--home` CLI flag in example scripts), the path is expanded and resolved to an absolute path. The library expects `nexatomTT.dll` to exist at `<home>/nexatomTT.dll`.

**2. `NEXATOMTT_HOME` environment variable**

```powershell
set NEXATOMTT_HOME=C:\nexatomtt-sdk-windows-x64
python python\examples\load_smoke.py
```

If no `home` parameter is passed, the wrapper checks the `NEXATOMTT_HOME` environment variable. If set, its value is used as the SDK home.

**3. Automatic directory traversal**

If neither an explicit `home` nor `NEXATOMTT_HOME` is available, the wrapper searches for `nexatomTT.dll` by walking the directory tree upward from the location of `_native.py`. The search visits:

| Candidate path | Layout |
|---|---|
| `<base>.parents[3]` | Installed app layout (`<app>/integrations/python/nexatomtt/_native.py`) |
| `<base>`, `<base>/..`, `<base>/../..`, … | SDK layout (DLL at any ancestor directory) |
| `<ancestor>/bin/` | Development build layout |
| `<ancestor>/build/bin/` | CMake build layout |

The first candidate directory that contains `nexatomTT.dll` is selected.

**Resolution logic (from source):**

```python
def find_default_home(start: Path | None = None) -> Path:
    env_home = os.environ.get("NEXATOMTT_HOME")
    if env_home:
        return Path(env_home).expanduser().resolve()

    base = (start or Path(__file__)).resolve()
    for candidate in _candidate_homes(base):
        if (candidate / "nexatomTT.dll").exists():
            return candidate

    # Fallback: installed layout or parent directory
    if len(base.parents) >= 4:
        return base.parents[3]
    return base.parent
```

**Error behavior.** If the resolved home does not contain `nexatomTT.dll`, the constructor raises `FileNotFoundError` with the searched path and a diagnostic message recommending the `NEXATOMTT_HOME` variable or `--home` flag.