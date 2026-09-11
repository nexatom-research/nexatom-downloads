## Runtime ↔ Bootloader Handoff Validation

This tutorial validates the complete lifecycle of transitioning the hardware from active runtime acquisition into the bootloader service environment, querying flash memory status, and transitioning back. Orchestrating this safe transition is a strict prerequisite for performing in-field firmware updates.

**Relevant script:**
*   `runtime_bootloader_handoff.py`

### Workflow

1.  **Boot into Runtime:** The script leverages the standard `open_runtime_device()` orchestrator to ensure the hardware's starting state is `RUNTIME`.
2.  **Halt Hardware Engines:** Before yielding to the bootloader, it is best practice to halt any running acquisition engines (e.g. TIHI, MFCO) by issuing `device.request_global_stop_all_modes()`.
3.  **Request Service Entry:** The host issues the `device.request_field_upgrade_service_entry()` command. In response, the runtime firmware flushes its pending USB queues, sets a handoff flag in persistent memory, and triggers a system reset.
4.  **Await Protocol Shift:** The Python script loops and polls `device.hardware_protocol_mode()` until the device reports `NEXATOM_HARDWARE_PROTOCOL_MODE_BOOTLOADER`.
    > **Failure Handling.** If the timeout expires without a successful mode transition, the script issues `device.clear_field_upgrade_service_entry_request()` to clear the sticky hardware flag and gracefully abort.
5.  **Refresh Slot Table:** Once the device successfully reaches `BOOTLOADER` mode, the script queries the flash memory state via `device.refresh_field_update_status()` and renders a formatted table of all available firmware slots.
6.  **Return to Runtime (Optional):** If the `--boot-back` flag is passed, the script evaluates the flash table, issues `device.boot_field_update_slot(slot)`, and invokes a discovery polling loop to catch the USB disconnect/re-enumerate cycle. It confirms success only when `device.hardware_protocol_mode()` reports `RUNTIME` mode again.

### Execution

To run the handoff validation and force a complete round-trip (Runtime $\rightarrow$ Bootloader $\rightarrow$ Runtime):

```powershell
python python\examples\runtime_bootloader_handoff.py --boot-back
```

**Expected Output:**

```text
Runtime firmware is ready; requesting bootloader service handoff.
Bootloader service handoff confirmed; refreshing slot table.
[Slot 0] VALID (default) | v1.0.0
[Slot 1] EMPTY
Booting runtime slot 0 to prove bootloader-to-runtime return path.
Reconnecting after boot-back and checking runtime mode.
Post-boot-back hardware protocol mode: RUNTIME
Runtime firmware is ready again after boot-back.
Runtime/bootloader handoff smoke complete.
```
