## Runtime ↔ Bootloader Handoff Validation

This tutorial validates the complete lifecycle of transitioning the hardware from active runtime acquisition into the bootloader service environment, querying flash memory status, and transitioning back. Orchestrating this safe transition is a strict prerequisite for performing in-field firmware updates.

**Relevant script:**
*   `runtime_bootloader_handoff.py`

### Workflow

1.  **Boot into Runtime:** The script leverages the standard `open_runtime_device()` orchestrator to ensure the hardware's starting state is `RUNTIME`.
2.  **Halt Hardware Engines:** Before yielding to the bootloader, it is best practice to halt any running acquisition engines (e.g. TIHI, MFCO) by issuing `device.request_global_stop_all_modes()`.
3.  **Request Service Entry:** Call `device.request_field_upgrade_service_entry()`. The native API and installed firmware handle the model-specific transition; the application does not implement PS/MicroBlaze reset logic.
4.  **Complete Service Discovery:** Call `device.refresh_field_update_status()` immediately after the request. Native retains the transport and performs the service exchange. Polling a cached protocol-mode getter alone cannot initiate that exchange.
    > **Failure Handling.** On a failed refresh the example attempts `clear_field_upgrade_service_entry_request()` and reports both the original error and any cleanup failure. It does not treat a failed handoff as a usable service connection.
5.  **Inspect Slot Table:** Verify the returned mode is `BOOTLOADER` and inspect the refreshed slot entries. No flash image is written by this tutorial.
6.  **Return to Runtime (Optional):** With `--boot-back`, select a valid slot and call `device.boot_field_update_slot(slot)`. Native boot establishes runtime readiness on the same handle. The example then deliberately closes/reopens for an additional lifecycle check; normal measurement clients do not need that extra reopen loop.

### Execution

To run the handoff validation and force a complete round-trip (Runtime $\rightarrow$ Bootloader $\rightarrow$ Runtime):

```powershell
python python/examples/runtime_bootloader_handoff.py --home . --timeout-ms 20000 --boot-back
```

**Illustrative output:** slot count, versions and selected slot depend on the board.

```text
Runtime firmware is ready; requesting bootloader service handoff.
Bootloader service handoff confirmed; slot table refreshed.
[Slot 0] VALID (default) | v1.0.0
[Slot 1] EMPTY
Booting runtime slot 0 to prove bootloader-to-runtime return path.
Reconnecting after boot-back and checking runtime mode.
Post-boot-back hardware protocol mode: RUNTIME
Runtime firmware is ready again after boot-back.
Runtime/bootloader handoff smoke complete.
```

Omit `--boot-back` to leave the instrument in service. An explicit `--boot-slot` must identify a valid existing image. This round trip applies to bootloader-equipped instruments; a legacy runtime without service-entry support is not expected to pass it. See [firmware management](../01_getting_started/1_4_firmware.md) before performing any persistent operation.
