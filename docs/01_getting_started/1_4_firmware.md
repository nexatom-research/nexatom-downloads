# 1.4 Firmware management

Preview.7 supplies firmware service tools, **not a firmware image**. Ordinary acquisition can boot an existing valid runtime using native startup. Only load an image supplied for the exact instrument and intended application.

## Firmware image format

The loader's filename convention is four ASCII alphanumeric application characters, an underscore, a decimal version and `.bin`, for example `BOOT_001.bin`. This is an illustrative name, not a bundled or recommended image. Do not infer hardware compatibility from the filename alone or invent a host-side binary header.

## Firmware manifest

Retain the image supplier's identity, target compatibility, size and checksum information. Package identity and image compatibility are different checks. Do not reuse hashes or sizes from the preview.6 example image as requirements for a new image.

## Field update workflow

1. Connect the intended device and enter its firmware service through the public API when needed.
2. Refresh status and inspect the reported slot table; do not hard-code the number of slots.
3. Select the intended image and target slot, preserving a known-good image where possible.
4. Request the guarded load and check its result/progress.
5. Boot the selected valid slot on the same handle and verify runtime identity/profile.

Setting the default slot is a separate persistent operation. It is not required for a one-time boot or a capture.

## Loading a firmware image

```sh
# Inspect required options and overwrite confirmations before any write.
python python/examples/field_update_e2e.py --help
```

Use the [field-update tutorial](../03_tutorials/3_7_end_to_end_firmware_field_update.md) for a concrete guarded command. Do not bypass overwrite checks, add fixed sleeps as readiness proof, or close and rediscover a new handle after native boot.

## Slot management

Use `refresh_field_update_status`, `boot_field_update_slot` and, only when intended, `set_field_update_default_slot`. A status refresh reports available state; it is not a promise that every image has just been fully CRC-checked. Empty/corrupt slots cannot be selected as usable runtimes.

## Safety precautions

**Firmware loading changes persistent storage. Keep power and USB connected while writing, and use only a compatible image.** A recovery path depends on the actual service firmware and remaining valid images; this manual does not guarantee recovery from every interruption.

[Getting started](index.md) · [Runtime handoff](../02_hardware/2_12_runtime_handoff.md)
