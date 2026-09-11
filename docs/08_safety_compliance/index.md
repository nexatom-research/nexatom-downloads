# Safety, Compliance, and Legal

### [License](index.md#license)

The NexatomTT SDK is provided under a non-exclusive license strictly for use with official Nexatom instrumentation hardware. Reverse engineering the proprietary USB communication protocol or utilizing the SDK to interface with non-Nexatom hardware is strictly prohibited.

For the complete and legally binding terms of use, refer to the `LICENSE.txt` file included in the root directory of the SDK distribution.

### [Third-Party Notices](index.md#third-party-notices)

To provide high-performance hardware interfacing and data processing capabilities, the NexatomTT SDK distributes and links against several third-party libraries:

*   **FTDI D3XX Runtime:** Distributed under the proprietary FTDI Chip software license.
*   **GCC Runtime Libraries:** (`libgcc`, `libstdc++`) Distributed under the GNU GPL v3.0 with the GCC Runtime Library Exception, which permits linking with proprietary and closed-source software without viral licensing effects.
*   **MinGW-w64 winpthreads:** Distributed under permissive open-source licenses.
*   **HDF5 (1.14.6):** Utilized for processed data exports, distributed under the HDF Group / NCSA BSD-style license.

For exact copyright attribution, maintainer lists, and the full text of these licenses, refer to the `THIRD_PARTY_NOTICES.md` file included in the SDK root.

### [Firmware Safety Notice](index.md#firmware-safety-notice)

> [!CAUTION]
> **Field Update Interruption Risk**
> When executing a firmware field update via the SDK (or the `field_update_e2e.py` tool), **do not disconnect the USB cable or interrupt power to the device** until the `FINALIZE` phase successfully completes.

*   **Image Validation:** Only flash firmware images that adhere to the `<NAME>_<VERSION>.bin` convention and have been explicitly provided by Nexatom for your exact hardware model (e.g., UTT810).
*   **Boot Failures:** Loading a corrupt, truncated, or incompatible firmware image into the `default` boot slot will prevent the device from booting into `RUNTIME` mode. In this event, the device will permanently fallback to `BOOTLOADER` mode upon power-up, requiring you to re-flash a valid image to restore acquisition capabilities.
