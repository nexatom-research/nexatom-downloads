# Field Update / Bootloader

These endpoints provide low-level access to the internal FPGA bootloader state machine. They allow host applications to partition flash memory, validate CRC32 checksums, and perform in-field bitstream updates over USB.

*Note: For standard firmware updates, we highly recommend utilizing the pre-built Python deployment utility (see Chapter 4). These C endpoints are exposed strictly for OEM developers writing custom, embedded updating wrappers.*

### Function Reference

| Function | Parameters (In/Out) | Returns | Description |
| :--- | :--- | :--- | :--- |
| `nexatom_tt_request_field_upgrade_service_entry` | `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Sets an EEPROM flag to force the FPGA bootloader to remain in service mode upon the next power cycle, rather than jumping to the runtime image. |
| `nexatom_tt_clear_field_upgrade_service_entry_request`| `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Clears the sticky EEPROM service entry flag. |
| `nexatom_tt_load_field_update_image` | `[In] nexatom_tt_handle device`<br>`[In] const nexatom_field_update_request_t* request`<br>`[In] nexatom_field_update_progress_callback_t progress`<br>`[In] void* user_data` | `nexatom_error_code_t` | Synchronously reads a new `.bin` firmware file from disk and uploads it to the FPGA flash memory block-by-block, invoking the provided progress callback. |
| `nexatom_tt_refresh_field_update_status` | `[In] nexatom_tt_handle device`<br>`[Out] nexatom_field_update_status_t* status`<br>`[Out] nexatom_field_update_slot_info_t* slots`<br>`[In] size_t max_slots`<br>`[Out] size_t* out_slot_count` | `nexatom_error_code_t` | Probes the bootloader to extract the flash memory map, calculating the CRC32 validity of all firmware images currently stored. |
| `nexatom_tt_set_field_update_default_slot`| `[In] nexatom_tt_handle device`<br>`[In] uint32_t slot_index` | `nexatom_error_code_t` | Commands the bootloader to mark a specific flash partition slot as the default boot image on power-on. |
| `nexatom_tt_set_field_update_default_slot_with_status`| `[In] nexatom_tt_handle device`<br>`[In] uint32_t slot_index`<br>`[Out] nexatom_field_update_status_t* status`<br>`[Out] nexatom_field_update_slot_info_t* slots`<br>`[In] size_t max_slots`<br>`[Out] size_t* out_slot_count` | `nexatom_error_code_t` | Atomic operation that sets the default boot slot and instantly returns the refreshed slot table metadata. |
| `nexatom_tt_boot_field_update_slot` | `[In] nexatom_tt_handle device`<br>`[In] uint32_t slot_index` | `nexatom_error_code_t` | Issues the command to vector the microprocessor program counter into the designated flash slot, immediately starting the runtime firmware. |

### Data Structures: `nexatom_field_update_request_t`

When invoking a firmware load, developers must construct this request block to define the target slot and post-load behavior. To ensure cross-version ABI compatibility, `struct_size` must always be initialized to `sizeof(nexatom_field_update_request_t)`.

| Field | Type | Description |
| :--- | :--- | :--- |
| `struct_size` | `uint32_t` | ABI verification. Must equal the byte size of the struct. |
| `slot_index` | `uint32_t` | Target flash partition (0-3). |
| `image_path` | `const char*` | Absolute file path to the native NexatomTT `.bin` update file. |
| `startup_ack_response_code` | `uint32_t` | Legacy bootloader parameter. Leave as `0`. |
| `set_default_after_load` | `uint8_t` | Pass `1` to automatically mark this slot as the boot default on success. |
| `boot_after_load` | `uint8_t` | Pass `1` to immediately jump execution into this slot on success. |
| `_padding0` | `uint8_t[2]` | FFI alignment padding. |
| `request_flags` | `uint32_t` | Advanced override flags (default `0`). |
| `reserved` | `uint8_t[12]` | Reserved for future ABI extensions. Must be zeroed. |

### C Example: Flashing an Update

```c
// 1. Prepare the request payload
nexatom_field_update_request_t req = {0};
req.struct_size = sizeof(nexatom_field_update_request_t);
req.slot_index = 1;
req.image_path = "/usr/local/firmware/nexatom_v2_1.bin";
req.set_default_after_load = 1;
req.boot_after_load = 1;

// 2. Define a simple progress callback
void my_progress_cb(nexatom_field_update_progress_t progress, void* user) {
    printf("Flashing Phase %d... %d%%\n", progress.phase, progress.percent);
}

// 3. Execute the synchronous upload block (will block for ~5-15 seconds)
nexatom_error_code_t err = nexatom_tt_load_field_update_image(
    my_device, 
    &req, 
    my_progress_cb, 
    NULL
);

if (err == 0) {
    printf("Firmware update successful! FPGA rebooting.\n");
}
```
