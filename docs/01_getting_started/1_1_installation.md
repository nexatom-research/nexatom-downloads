# Installation Guide

### [System requirements](1_1_installation.md#system-requirements)

| Requirement | Detail |
|---|---|
| Operating system | Windows 10 or later, 64-bit; or Linux x86-64 matching the release's runtime requirements |
| CPU architecture | x86-64 (AMD64) |
| USB | USB 3 port and suitable data cable for acquisition |
| FTDI D3XX driver | Required for USB transport to the UTT810 |
| Python (optional) | CPython 3.10 or later (for Python wrapper and examples) |
| Disk space | Space for the extracted SDK and your recordings; raw captures can be much larger than the SDK |

No compiler, build system, or `pip install` step is required to run the headless Python examples. The Python wrapper uses the standard library. The optional live plotting example (`tihi_mfco_matplotlib.py`) requires `matplotlib`. Building the C/C++ examples requires a compiler and CMake; see [Programming Languages](1_3_programming.md).

Download the **Windows x64 ZIP** or **Linux x64 archive** from the [SDK release page](https://www.nexatom.in/downloads/utt810/sdk-preview/). Extract the complete package. Both packages expose the same C API and Python operations. On Linux, use `python3` in the commands below if `python` is unavailable.

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

> **Note.** The Windows SDK ships `FTD3XXWU.dll` alongside `nexatomTT.dll`. This is the user-mode runtime that the native library links against at load time. The Windows device driver must also be installed. Keep the bundled runtime with its SDK; see `LICENSE.txt` and `THIRD_PARTY_NOTICES.md` for the supplied terms and notices.

#### Linux device access

The Linux SDK includes the FTDI D3XX 1.1.8 user-space library, `libftd3xx.so`. Keep it beside `libnexatomTT.so` and `libnexatomTT.so.1`. A Windows driver installer is not used on Linux.

An administrator can install the supplied USB-access rule from the extracted SDK:

```sh
sudo install -m 644 drivers/linux/51-ftd3xx.rules /etc/udev/rules.d/51-ftd3xx.rules
sudo udevadm control --reload-rules
```

Then reconnect the device's data USB and run examples as your ordinary user. Installing the rule is the only step that needs root; do not run acquisition programs with `sudo`, which leaves files owned by root and hides a permission problem. The supplied vendor rule grants `MODE="0666"` access to its listed FTDI devices (vendor `0403`, including the FT601's product `601f`); a managed laboratory may use a group-based rule instead, for example `MODE="0660", GROUP="plugdev"` with the users added to that group. Without a rule, discovery can list the board while opening it fails for a non-root user. If discovery succeeds but opening fails, check USB permissions and whether another application owns the device.

WSL additionally requires USB forwarding from Windows into the intended distribution. Native Linux with a directly connected device does not require this forwarding step. Finish acquisition and close the device before changing which operating system owns it.

---

### [SDK package contents](1_1_installation.md#sdk-package-contents)

Extract the SDK archive. The resulting directory contains the following files:

| File | Platform | Role |
|---|---|---|
| `nexatomTT.dll` | Windows | Core native library, exporting the public C ABI |
| `FTD3XXWU.dll` | Windows | FTDI D3XX user-mode runtime |
| `libnexatomTT.so`, `libnexatomTT.so.1` | Linux | Core shared library and its versioned name |
| `libftd3xx.so` | Linux | FTDI D3XX user-space runtime |
| `manifest.json` | Both | Package version, native identity and file checksums |
| `README.md` | Both | Package overview |
| `LICENSE.txt`, `THIRD_PARTY_NOTICES.md` | Both | SDK terms and third-party notices |

| Directory | Contents |
|---|---|
| `include/` | Standalone `nexatomtt_c_api.h` public C ABI header |
| `lib/` | Windows MinGW import library, `nexatomTT.dll.a` |
| `python/nexatomtt/` | Python loader, device controls, structures, analysis helpers and runtime helper |
| `python/examples/` | Commented measurement, saving, plotting and service examples |
| `examples/sdk/` | Standalone C/C++ project with shared acquisition code |
| `firmware/` | Current firmware catalogue: `firmware_manifest.json` (schema 2) and the images it lists, `Z080_004.bin` (UTT810) and `K168_004.bin` (UTT160810) in this SDK |
| `drivers/linux/` | Linux USB permission rule `51-ftd3xx.rules` (Linux package) |
| `licenses/` | Dependency licences `HDF5-COPYING`, `FTDI-README.pdf` and `FTDI-ftd3xx.h` (Linux package); the Windows package carries its notices in `THIRD_PARTY_NOTICES.md` |
| `docs/` | Packaged C API developer guide |

The `firmware/` folder holds the current catalogue for both instruments; later catalogues are published as `firmware-catalog-N` releases (see [Firmware catalogues](1_4_firmware.md#firmware-catalogues)). This SDK does not require a firmware download for ordinary acquisition when the instrument already has a usable runtime. See [Firmware Management](1_4_firmware.md) for intentional image updates.

**DLL co-location rule.** Both DLL files must reside in the same directory. The compiler runtime and HDF5 are linked into `nexatomTT.dll`, so no other runtime DLLs are needed. The Python wrapper loads `nexatomTT.dll` by absolute path from the SDK root and calls `os.add_dll_directory()` so that Windows can resolve the FTDI runtime DLL at load time. Moving one DLL to a separate directory will cause `OSError` on import.

```
nexatomtt-sdk-windows-x64/
├── nexatomTT.dll            ← loaded by ctypes.CDLL()
└── FTD3XXWU.dll             ← resolved via os.add_dll_directory()
```

For C/C++ consumers, ensure the SDK root is on the DLL search path (e.g., place the executable in the SDK root, or add the SDK root to the `PATH` environment variable).

On Linux, keep the shared libraries in their extracted layout. The supplied CMake example project links the selected SDK and sets its runtime search path. Python loads the library from the selected SDK home. The Linux package uses the system C/C++ runtime; consult the release requirements when choosing the host distribution.

---

### [Verifying the installation](1_1_installation.md#verifying-the-installation)

Two verification steps confirm the SDK is correctly installed: a DLL load smoke test and a version info query. Neither requires connected hardware.

#### DLL load smoke test

From a terminal opened at the SDK root:

```powershell
python python/examples/load_smoke.py --home .
```

Output has this form (the native version string depends on the package):

```
Loaded NexatomTT <native-version>
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
python python/examples/version_info.py
```

This script calls both `nexatom_tt_get_version()` and `nexatom_tt_get_library_info()`:

```python
from nexatomtt import NexatomLibrary

lib = NexatomLibrary()
print(lib.version())       # Native library version, not the SDK archive tag
print(lib.library_info())  # Package/native build identification
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

### [SDK home discovery](1_1_installation.md#sdk-home-discovery)

The Python wrapper must locate the directory containing `nexatomTT.dll` on Windows or `libnexatomTT.so` on Linux. This directory is called the **SDK home**. The `NexatomLibrary` constructor resolves it through the following precedence chain:

**1. Explicit `home` parameter**

```python
lib = NexatomLibrary(home=r"C:\nexatomtt-sdk-windows-x64")
```

When `home` is supplied (via constructor argument or `--home` CLI flag in example scripts), the path is expanded and resolved to an absolute path. The platform's native library must exist at that location. For Linux, an example is `NexatomLibrary(home="/opt/nexatomtt-sdk")`.

**2. `NEXATOMTT_HOME` environment variable**

```powershell
$env:NEXATOMTT_HOME = 'C:\nexatomtt-sdk-windows-x64'
python python/examples/load_smoke.py
```

If no `home` parameter is passed, the wrapper checks the `NEXATOMTT_HOME` environment variable. If set, its value is used as the SDK home.

In a Linux shell the equivalent is `export NEXATOMTT_HOME=/opt/nexatomtt-sdk`.

**3. Automatic directory traversal**

If neither an explicit `home` nor `NEXATOMTT_HOME` is available, the wrapper searches for the platform library by walking the directory tree upward from the location of `_native.py`. The search visits:

| Candidate path | Layout |
|---|---|
| `<base>.parents[3]` | Installed app layout (`<app>/integrations/python/nexatomtt/_native.py`) |
| `<base>`, `<base>/..`, `<base>/../..`, … | SDK layout (DLL at any ancestor directory) |
| `<ancestor>/bin/` | Development build layout |
| `<ancestor>/build/bin/` | CMake build layout |

The first candidate directory containing the platform library is selected. Use explicit `home` when several SDK versions are present, so it is clear which package an experiment uses.

**Resolution logic (simplified):**

```python
def find_default_home(start: Path | None = None) -> Path:
    env_home = os.environ.get("NEXATOMTT_HOME")
    if env_home:
        return Path(env_home).expanduser().resolve()

    base = (start or Path(__file__)).resolve()
    for candidate in _candidate_homes(base):
        if (candidate / _native_library_name()).exists():
            return candidate

    # Fallback: installed layout or parent directory
    if len(base.parents) >= 4:
        return base.parents[3]
    return base.parent
```

**Error behavior.** If the resolved home does not contain the platform library, the constructor raises `FileNotFoundError` with the searched path. A missing dependency raises `OSError`. A mismatch between the supplied manifest and library raises a verification error. Keep the package intact rather than combining a new Python folder with an older DLL.

For your own script, put it beside the `nexatomtt` package in the SDK's `python/` directory, or add that directory to `PYTHONPATH`. `NEXATOMTT_HOME` selects the native library; `PYTHONPATH` controls Python module imports. The supplied examples set their import path themselves.

---

### [Troubleshooting](1_1_installation.md#troubleshooting)

| Symptom | Cause and fix |
|---|---|
| A Windows program exits at start with code `0xc0000135` (`STATUS_DLL_NOT_FOUND`) | A DLL the program needs was not found. Keep `nexatomTT.dll` and `FTD3XXWU.dll` next to the program, or put the SDK root on `PATH`. The supplied CMake project copies both beside its executables. |
| Python raises `OSError` when loading the library | The same cause: one of the two DLLs was moved away from the SDK root, or `NEXATOMTT_HOME` names another folder. Keep the extracted package intact. |
| Windows lists no device | The FTDI D3XX driver is not installed, or the board is not connected to a working USB 3.0 port. Check Device Manager for the FTDI D3XX device. |
| Linux lists the board but opening it fails | USB permissions. Install `drivers/linux/51-ftd3xx.rules` as shown above, reload the rules and reconnect the board, then run as your ordinary user. |
| Opening fails with `... already open ...` | Another program or handle owns that board. Close it first; a board can be open in one handle at a time. |
