# Safety, Compliance, and Legal

### [License](index.md#license)

The NexatomTT SDK is licensed for applications operating Nexatom hardware. The package's terms define permitted use/redistribution, restrictions and applicable exceptions; this summary does not replace them.

For the complete and legally binding terms of use, refer to the `LICENSE.txt` file included in the root directory of the SDK distribution.

### [Third-Party Notices](index.md#third-party-notices)

To provide high-performance hardware interfacing and data processing capabilities, the NexatomTT SDK distributes and links against several third-party libraries:

*   **FTDI D3XX Runtime:** Distributed under the proprietary FTDI Chip software license.
*   **GCC Runtime Libraries:** (`libgcc`, `libstdc++`) Supplied under their applicable GNU GPL terms with the GCC Runtime Library Exception; retain the full notices and exception text.
*   **MinGW-w64 winpthreads:** Distributed under permissive open-source licenses.
*   **HDF5 (1.14.6):** Utilized for processed data exports, distributed under the HDF Group / NCSA BSD-style license.

For exact copyright attribution, maintainer lists, and the full text of these licenses, refer to the `THIRD_PARTY_NOTICES.md` file included in the SDK root.

### [Firmware Safety Notice](index.md#firmware-safety-notice)

> [!CAUTION]
> **Field Update Interruption Risk**
> When executing a firmware field update via the SDK (or `field_update_e2e.py`), **do not disconnect USB or interrupt power while writing. Check the final successful load/verification result**; the progress API uses its documented COMPLETED/FAILED phases, not a FINALIZE phase.

*   **Image Validation:** Only flash firmware images that adhere to the `<NAME>_<VERSION>.bin` convention and have been explicitly provided by Nexatom for your exact hardware model (e.g., UTT810).
*   **Boot Failures:** A corrupt or incompatible default image can prevent runtime boot. Recovery depends on the service firmware and remaining valid slots; do not assume recovery is guaranteed. Preserve a known-good image where possible.

The SDK bundles the current firmware catalogue in its `firmware/` folder (manifest plus images); later catalogues are published as `firmware-catalog-N` releases, which the Nexatom app finds through `https://downloads.nexatom.in/firmware/latest.json`. Load only an image whose manifest entry names your model, and check its size and SHA-256 against that manifest. Keep the complete platform package, FTDI/vendor licenses and notices. Linux includes the bundled FTDI D3XX userspace library and optional USB permission rule; Windows includes `FTD3XXWU.dll` and uses its matching FTDI driver. The compiler runtime is linked into the native library, so no separate runtime DLLs are needed.
