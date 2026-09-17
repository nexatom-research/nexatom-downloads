# 2.1 Operating conditions

## USB data connection

Use the instrument's specified USB cable, port and power arrangement, plus the [platform driver setup](../01_getting_started/1_1_installation.md). This manual does not qualify USB 2 fallback, hub power budgets or maximum sustained acquisition rates. Do not run competing clients against one device.

## Device discovery and enumeration

`discover_devices()` returns selection records. Preserve the **full FT601 serial string** when identifying a unit. It is programmable USB identity, not proof of an immutable or globally unique serial. A device description does not establish product model or firmware capability. Duplicate/ambiguous identifiers need an intentional selection or corrected device provisioning; do not silently choose a different unit.

For initial examples connect one intended device: the packaged primary templates select and print the first discovered unit. For a test PC with both boards connected, inspect enumeration and select one unit for each sequential run. Concurrent acquisition is outside this manual's qualified workflow. Do not treat discovery index/connection ID as a persistent physical-port identifier.

## Connection management and state machine

Create one handle from the selected record and call `connect_runtime` (Python `open_runtime_device` is the context helper). Native startup waits for runtime/profile readiness and may boot an existing valid slot. Inspect failures rather than assuming that an open USB handle authorizes measurements.

## Hardware protocol modes

UNKNOWN, BOOTLOADER and RUNTIME describe protocol state. Runtime state alone is not a substitute for an authorized profile. Read `get_device_profile()` and inspect control authorization, service state, `effective_public_tdc_mask`, output-mode mask and feature flags before configuring.

Current product IDs include `0x4E580001` (UTT160810) and `0x4E580002` (UTT810). Let native resolve identity and limits; do not implement firmware-layout or model dispatch in acquisition code. Physical TDC count, public channel mask and the legacy callback array extent have different meanings.

[Device operation](index.md) · [Runtime startup](../06_in_depth_guides/6_6_bootloader_first_device_startup.md)
