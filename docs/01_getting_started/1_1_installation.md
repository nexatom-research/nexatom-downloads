# 1.1 Installation guide

## System requirements

Choose the published Windows x64 or Linux x86-64 FTDI package matching your host. Use a 64-bit Python interpreter compatible with the packaged Python guide; core examples use the standard library. Matplotlib is optional for the plotting tutorial. C/C++ users also need the compiler and CMake described in `examples/sdk/README.md` inside the package.

Use the instrument's specified USB connection and power arrangement. This SDK manual does not qualify USB 2 fallback or provide electrical limits for every instrument.

## FTDI D3XX driver installation

On Windows, install the FTDI D3XX device driver appropriate to the instrument. The SDK's `FTD3XXWU.dll` is the application runtime; copying it is not a kernel-driver installation.

On Linux, keep the bundled `libftd3xx.so` beside `libnexatomTT.so`. If USB access is denied, an administrator may install the supplied rule:

```sh
# Optional administrator setup on a system running udev.
sudo install -m 0644 drivers/linux/51-ftd3xx.rules /etc/udev/rules.d/51-ftd3xx.rules
sudo udevadm control --reload-rules
# Reconnect the instrument, then run acquisitions as your ordinary user.
```

The supplied vendor rule grants access to all local users for its listed devices (`MODE="0666"`). A site administrator can instead apply a suitable group-based rule. WSL additionally needs Windows USB forwarding; one OS/client owns the device at a time.

**Keep the SDK's matching FTDI and compiler runtimes together. Do not replace individual libraries to troubleshoot a capture or run competing clients against the same device.**

## SDK package contents

Keep the complete extracted directory, including:

| Path | Purpose |
| --- | --- |
| `nexatomTT.dll` or `libnexatomTT.so` | Native API |
| Adjacent runtime libraries | Platform-specific dependencies |
| `include/nexatomtt_c_api.h` | Public C declarations, also used by C++ |
| `python/nexatomtt/` | Python binding |
| `python/examples/` | Runnable Python workflows |
| `examples/sdk/` | Standalone C/C++ consumer project |
| `docs/`, license files and notices | Package-specific reference and terms |
| `manifest.json` | Package identity |

The SDK does not bundle firmware images. Follow the package's own README for platform-specific paths and compiler details.

## Verifying the installation

Run from the extracted SDK directory:

```sh
# Load the library and print native version/build identity; no device is opened.
python python/examples/version_info.py --home .
# Show acquisition options without starting hardware.
python python/examples/processed_acquisition.py --help
```

A successful library load does not prove USB permissions, runtime readiness or acquisition. Check those with the [quick start](1_2_quick_start.md). The native version string is separate from the SDK's preview.8 release label.

## SDK home discovery

Prefer an explicit `--home` for examples. Applications can pass `NexatomLibrary(home=...)`. If using the supported environment override in PowerShell:

```powershell
# Point the loader to this complete extraction, not an individual DLL.
$env:NEXATOMTT_HOME = (Get-Location).Path
```

[Getting started](index.md) · [Quick start](1_2_quick_start.md)
