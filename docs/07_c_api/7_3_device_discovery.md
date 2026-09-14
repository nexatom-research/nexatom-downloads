# Device Discovery and Lifecycle

These endpoints manage scanning the host's USB controller for attached NexatomTT instruments and safely allocating (and destroying) the opaque `nexatom_tt_handle`.

*Note: Allocating the handle via `nexatom_tt_create` does **not** open the physical USB endpoints. You must call `nexatom_tt_connect` (see Section 7.4) after creation.*

### Function Reference

| Function | Parameters (In/Out) | Returns | Description |
| :--- | :--- | :--- | :--- |
| `nexatom_tt_discover_devices` | `[Out] nexatom_tt_info_t* devices`<br>`[In] size_t max_devices`<br>`[Out] size_t* num_devices` | `nexatom_error_code_t` | Scans the USB bus and populates the user-provided array with identification metadata for all detected instruments. |
| `nexatom_tt_create` | `[In] const nexatom_tt_info_t* device_info`<br>`[Out] nexatom_tt_handle* device` | `nexatom_error_code_t` | Allocates the internal C++ host structures for a specific device and returns an opaque pointer handle. |
| `nexatom_tt_destroy` | `[In] nexatom_tt_handle device` | `void` | Frees all host memory allocated for the handle. Must be called exactly once per handle before process exit to prevent memory leaks. |

### Data Structures: `nexatom_tt_info_t`

When discovering devices, the SDK returns a static metadata block for each instrument. All strings are guaranteed to be UTF-8 encoded and null-terminated. `NEXATOM_MAX_STRING_LENGTH` is explicitly defined as `256` bytes.

| Field | Type | Description |
| :--- | :--- | :--- |
| `serial_number` | `char[256]` | Unique factory serial number string (e.g., `"NTT-10045"`). |
| `firmware_version` | `char[256]` | Running FPGA bitstream version. |
| `hardware_version` | `char[256]` | PCB revision identifier. |
| `device_name` | `char[256]` | Human-readable product name string. |
| `connection_type` | `char[256]` | Underlying transport layer (typically `"FTDI"` or `"Mock"`). |
| `connection_id` | `char[256]` | OS-level unique hardware identifier or COM port mapping. |

### C Example: Enumerating and Creating a Handle

```c
nexatom_tt_info_t devices[4];
size_t num_found = 0;
nexatom_tt_handle my_device = NULL;

// 1. Scan the USB bus (up to 4 devices)
nexatom_error_code_t err = nexatom_tt_discover_devices(devices, 4, &num_found);
if (err == 0 && num_found > 0) {
    printf("Found %zu NexatomTT instrument(s)!\n", num_found);
    printf("First device Serial: %s\n", devices[0].serial_number);
    
    // 2. Allocate the handle memory for the first device
    err = nexatom_tt_create(&devices[0], &my_device);
    if (err == 0) {
        printf("Handle successfully allocated.\n");
        
        // ... (proceed to connect and measure) ...
        
        // 3. Teardown
        nexatom_tt_destroy(my_device);
    }
}
```
