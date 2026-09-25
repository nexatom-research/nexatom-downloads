# Library Version and Error Handling

These foundational functions operate entirely independently of any hardware handles (`nexatom_tt_handle`). They are safe to call before `nexatom_tt_create` is invoked.

Developers should use these endpoints to verify SDK compatibility at runtime and to extract human-readable exception strings whenever a native API call returns a negative `nexatom_error_code_t`.

### Function Reference

| Function | Parameters (In/Out) | Returns | Description |
| :--- | :--- | :--- | :--- |
| `nexatom_tt_get_version` | *None* | `const char*` | Returns the semantic version string (e.g., `"1.2.0"`) of the native C++ library. The returned pointer is statically allocated and must not be freed by the host. |
| `nexatom_tt_get_library_info` | `[Out] char* info_buffer`<br>`[In] size_t buffer_size` | `nexatom_error_code_t` | Writes the loaded library's build information into the caller's buffer. Retain it with the SDK package identity; the native version and SDK release label are separate. |
| `nexatom_tt_get_error_message` | `[In] nexatom_error_code_t error_code` | `const char*` | Returns a static, human-readable string representing the specified enum error code (e.g., `"NEXATOM_ERROR_TIMEOUT"`). |
| `nexatom_tt_get_last_error_message` | *None* | `const char*` | Returns the current process-global detailed native diagnostic. Copy it promptly before another operation replaces it; do not free the returned pointer. |

### Error Handling Example (C/C++)

```c
// Fragment: device is a successfully created handle for the selected instrument.
nexatom_error_code_t status = nexatom_tt_connect_runtime(device, 20000);

if (status != 0) {
    // 1. Get the generic enum string
    const char* error_name = nexatom_tt_get_error_message(status);
    
    // 2. Get the detailed C++ exception context
    const char* detailed_reason = nexatom_tt_get_last_error_message();
    
    printf("Connection failed! [%s] Detail: %s\n", error_name, detailed_reason);
}
```

### Device protocol ABI (1.0 and 1.1)

The library version above is the host software. The instrument's runtime reports a separate **device protocol ABI**, the command/register protocol it speaks, as major (bits 31..16) and minor (bits 15..0). The resolved profile carries it in `nexatom_tt_device_profile_v1_t.software_interface_abi` (the field keeps its historical name).

| Constant | Value | Runtimes |
| :--- | :--- | :--- |
| `NEXATOM_TT_DEVICE_PROTOCOL_ABI_1_0` | `0x00010000` | Earlier UTT160810 images |
| `NEXATOM_TT_DEVICE_PROTOCOL_ABI_1_1` | `0x00010001` | Current images: `Z080_004` (UTT810) and `K168_004` (UTT160810) |

This SDK recognises runtime ABI 1.0 and 1.1 exactly. A minor revision adds records and registers without changing existing ones, but the SDK never masks off the minor: a revision it does not know is refused rather than assumed compatible. Original UTT810 units without a bootloader report no device protocol ABI and are recognised through their legacy contract.

```c
// Fragment: after nexatom_tt_connect_runtime() succeeded on my_device.
nexatom_tt_device_profile_v1_t profile = {0};
profile.struct_size = sizeof(profile);
profile.struct_version = NEXATOM_TT_DEVICE_PROFILE_V1_VERSION;
if (nexatom_tt_get_device_profile_v1(my_device, &profile) == NEXATOM_SUCCESS) {
    printf("Device protocol ABI %u.%u\n",
           (unsigned)(profile.software_interface_abi >> 16),
           (unsigned)(profile.software_interface_abi & 0xFFFFu));
}
```
