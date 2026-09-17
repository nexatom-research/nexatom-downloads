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
