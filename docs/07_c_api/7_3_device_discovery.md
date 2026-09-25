# Device Discovery and Lifecycle

These endpoints manage scanning the host's USB controller for attached NexatomTT instruments and safely allocating (and destroying) the opaque `nexatom_tt_handle`.

*Creating a handle does not open the USB session. Use `nexatom_tt_connect_runtime` for measurement readiness; `nexatom_tt_connect` remains available for explicit service/diagnostic workflows. See [connection](7_4_device_connection.md).*

### Function Reference

| Function | Parameters (In/Out) | Returns | Description |
| :--- | :--- | :--- | :--- |
| `nexatom_tt_discover_devices` | `[Out] nexatom_tt_info_t* devices`<br>`[In] size_t max_devices`<br>`[Out] size_t* num_devices` | `nexatom_error_code_t` | Scans the USB bus and populates the user-provided array with identification metadata for all detected instruments. It never opens, resets or reads a board, and it also lists boards that are already open. |
| `nexatom_tt_create` | `[In] const nexatom_tt_info_t* device_info`<br>`[Out] nexatom_tt_handle* device` | `nexatom_error_code_t` | Allocates the internal C++ host structures for the board named by `device_info->connection_id` and returns an opaque pointer handle. An empty `connection_id` means "the first free board" at connect. |
| `nexatom_tt_destroy` | `[In] nexatom_tt_handle device` | `void` | Frees all host memory allocated for the handle. Must be called exactly once per handle before process exit to prevent memory leaks. |

### Data Structures: `nexatom_tt_info_t`

When discovering devices, the SDK returns a static metadata block for each instrument. All strings are guaranteed to be UTF-8 encoded and null-terminated. `NEXATOM_MAX_STRING_LENGTH` is explicitly defined as `256` bytes.

| Field | Type | Description |
| :--- | :--- | :--- |
| `serial_number` | `char[256]` | FT601 USB serial, for service information only. Several boards can carry the same factory serial (`000000000001`), so it never selects, gates or verifies a board. |
| `firmware_version` | `char[256]` | Available version string; discovery alone may not know the runtime image. |
| `hardware_version` | `char[256]` | Available hardware string; use the resolved profile for authoritative identity. |
| `device_name` | `char[256]` | Human-readable product name string. |
| `connection_type` | `char[256]` | Underlying transport layer (typically `"FTDI"` or `"Mock"`). |
| `connection_id` | `char[256]` | The board's identity: `"usb:"` + the USB port path. Windows: the PnP location path of the FT601 device node, for example `usb:PCIROOT(0)#PCI(0801)#PCI(0004)#USBROOT(0)#USB(4)`. Linux: the sysfs USB device name, for example `usb:2-1.3`. It stays the same while the board stays in that port; moving the board to another port changes it. |

### C Example: Enumerating and Creating a Handle

```c
nexatom_tt_info_t devices[4];
size_t num_found = 0;
nexatom_tt_handle my_device = NULL;

// 1. Scan the USB bus (up to 4 devices)
nexatom_error_code_t err = nexatom_tt_discover_devices(devices, 4, &num_found);
if (err == 0 && num_found == 1) {
    printf("Found %zu NexatomTT instrument(s)!\n", num_found);
    printf("Board at %s (FT601 serial %s, information only)\n",
           devices[0].connection_id, devices[0].serial_number);

    // 2. This example requires one unambiguous board; with several, choose the
    //    entry whose connection_id names the intended USB port.
    err = nexatom_tt_create(&devices[0], &my_device);
    if (err == 0) {
        printf("Handle successfully allocated.\n");
        
        // ... (proceed to connect and measure) ...
        
        // 3. Teardown
        nexatom_tt_destroy(my_device);
    }
}
```

<a id="board-identity-and-several-boards"></a>

### Board identity and several boards

- **Select by port.** `nexatom_tt_connect()` and `nexatom_tt_connect_runtime()` open exactly the board at the handle's `connection_id`, or fail. They never fall back to another board. Treat the id as opaque text: compare it exactly and pass the discovered record to `nexatom_tt_create()`.
- **Empty id.** A handle created with an empty `connection_id` connects to the first free board and is bound to that board's port from then on.
- **Errors.** No board in that port: `Device not found: no FT601 at USB port …`. Board in use by this or another process: `… already open …`. The discovery record has no busy flag. The message is available through `nexatom_tt_get_last_error_message()` (process-global; read it right after the failing call) and the log.
- **Several boards.** Two or more boards can be open at the same time through separate handles, including boards that share an FT601 serial.
- **Moving a board.** Plugging a board into another USB port gives it another `connection_id`. Discover again after re-cabling.
- **Discovery is passive.** Discovery never opens, resets or reads a board, so it is safe to call while boards are in use. A board whose port path would not fit in 255 bytes is not listed (the SDK logs it); the path is never shortened.
