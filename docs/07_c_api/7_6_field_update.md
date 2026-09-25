# Field Update / Bootloader

These endpoints inspect existing firmware slots, load a compatible image and perform service/runtime transitions. They do not provide arbitrary flash partitioning. The SDK supplies tools but no firmware image; images come from the [firmware catalogue](../01_getting_started/1_4_firmware.md#firmware-catalogues).

For a complete guarded workflow use the [Python field-update tutorial](../03_tutorials/3_7_end_to_end_firmware_field_update.md). Custom C/C++ updaters must preserve the same preflight, overwrite and completion checks.

### Function Reference

| Function | Parameters (In/Out) | Returns | Description |
| :--- | :--- | :--- | :--- |
| `nexatom_tt_request_field_upgrade_service_entry` | `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Requests entry from runtime into resident firmware service on the same handle. |
| `nexatom_tt_clear_field_upgrade_service_entry_request`| `[In] nexatom_tt_handle device` | `nexatom_error_code_t` | Clears the service-entry request where supported; does not program FT601 EEPROM identity. |
| `nexatom_tt_load_field_update_image` | `[In] nexatom_tt_handle device`<br>`[In] const nexatom_field_update_request_t* request`<br>`[In] nexatom_field_update_progress_callback_t progress`<br>`[In] void* user_data` | `nexatom_error_code_t` | Synchronously reads a new `.bin` firmware file from disk and uploads it to the FPGA flash memory block-by-block, invoking the provided progress callback. |
| `nexatom_tt_refresh_field_update_status` | `[In] nexatom_tt_handle device`<br>`[Out] nexatom_field_update_status_t* status`<br>`[Out] nexatom_field_update_slot_info_t* slots`<br>`[In] size_t max_slots`<br>`[Out] size_t* out_slot_count` | `nexatom_error_code_t` | Refreshes reported status and slot metadata; does not imply full CRC revalidation of every image. |
| `nexatom_tt_set_field_update_default_slot`| `[In] nexatom_tt_handle device`<br>`[In] uint32_t slot_index` | `nexatom_error_code_t` | Commands the bootloader to mark a specific flash partition slot as the default boot image on power-on. |
| `nexatom_tt_set_field_update_default_slot_with_status`| `[In] nexatom_tt_handle device`<br>`[In] uint32_t slot_index`<br>`[Out] nexatom_field_update_status_t* status`<br>`[Out] nexatom_field_update_slot_info_t* slots`<br>`[In] size_t max_slots`<br>`[Out] size_t* out_slot_count` | `nexatom_error_code_t` | Sets the default slot and returns the resulting status snapshot. Initialize `status.struct_size`; the operation involves device communication and can fail or time out. |
| `nexatom_tt_boot_field_update_slot` | `[In] nexatom_tt_handle device`<br>`[In] uint32_t slot_index` | `nexatom_error_code_t` | Boots an existing valid slot and waits for native runtime readiness on the same handle. |

### Data Structures: `nexatom_field_update_request_t`

When invoking a firmware load, developers must construct this request block to define the target slot and post-load behavior. To ensure cross-version ABI compatibility, `struct_size` must always be initialized to `sizeof(nexatom_field_update_request_t)`.

| Field | Type | Description |
| :--- | :--- | :--- |
| `struct_size` | `uint32_t` | ABI verification. Must equal the byte size of the struct. |
| `slot_index` | `uint32_t` | Target slot selected from the reported slot table. |
| `image_path` | `const char*` | Absolute file path to the native NexatomTT `.bin` update file. |
| `startup_ack_response_code` | `uint32_t` | Legacy bootloader parameter. Leave as `0`. |
| `set_default_after_load` | `uint8_t` | Pass `1` to automatically mark this slot as the boot default on success. |
| `boot_after_load` | `uint8_t` | Pass `1` to immediately jump execution into this slot on success. |
| `_padding0` | `uint8_t[2]` | FFI alignment padding. |
| `request_flags` | `uint32_t` | Advanced override flags (default `0`). |
| `reserved` | `uint8_t[12]` | Reserved for future ABI extensions. Must be zeroed. |

### C Example: Flashing an Update

```c
#include <stdio.h>
#include "nexatomtt_c_api.h"

// C callbacks are defined at file scope, outside the function using them.
static void my_progress_cb(nexatom_field_update_progress_t progress, void* user) {
    (void)user;
    printf("Flashing phase %u... %u%%\n",
           (unsigned)progress.phase, (unsigned)progress.percent);
}

// The caller has checked model/image compatibility, refreshed the slot table,
// and obtained the user's intent to replace this particular slot if occupied.
// It continues to own the handle and must disconnect/destroy it on every exit.
static nexatom_error_code_t load_checked_image(
    nexatom_tt_handle device, uint32_t target_slot, const char* image_path) {
    nexatom_field_update_request_t req = {0};
    req.struct_size = sizeof(req);
    req.slot_index = target_slot;
    req.image_path = image_path; // Nexatom-provided image for this model.
    req.set_default_after_load = 0; // Keep the existing power-on default.
    req.boot_after_load = 1;        // Also wait for runtime entry after loading.

    // This call is synchronous. Keep USB/power connected until it returns.
    nexatom_error_code_t err = nexatom_tt_load_field_update_image(
        device, &req, my_progress_cb, NULL);
    if (err != NEXATOM_SUCCESS) {
        fprintf(stderr, "Update failed: %s\n", nexatom_tt_get_last_error_message());
        return err;
    }
    printf("Load/boot completed; verify the expected runtime profile.\n");
    return NEXATOM_SUCCESS;
}
```

The path string and progress callback context must remain valid until the load returns. A percentage of 100 alone does not establish a successful load/boot: inspect the API return code and final progress phase. Preserve the existing valid image until the replacement has been verified; setting a new default is a separate persistent choice.
