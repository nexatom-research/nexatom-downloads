## Realtime CPS and Telemetry Streaming

Once the device is booted into the runtime firmware, the host can configure it to emit realtime diagnostics and event statistics. This tutorial demonstrates how to register Python callbacks and correctly sequence the hardware enable flags to receive a continuous stream of CPS (Counts Per Second) and telemetry packets.

**Relevant script:**
*   `ftdi_realtime_smoke.py`

### Workflow

1.  **Boot into Runtime:** The script leverages the `open_runtime_device()` orchestrator to acquire a valid `NexatomDevice` handle (as detailed in Section 3.2).
2.  **Register Callbacks:** Python functions are defined and passed to the native SDK via `device.set_count_rate_callback()` and `device.set_telemetry_callback()`. When the native background thread receives and decodes a relevant packet, these Python functions are invoked asynchronously with `NexatomCpsData` and `NexatomTelemetryData` structures.
3.  **Enable Hardware Subsystems:** The hardware modules are powered up and configured:
    *   `device.enable_system(True)`: Activates the primary FPGA data shuffler.
    *   `device.enable_telemetry(True)` / `device.set_telemetry_mode(2)`: Enables periodic 1 Hz telemetry emission.
    *   `device.set_cps_period_selector(0)`: Configures the CPS integration window to 1000 ms.
4.  **Route Output:** The final step calls `device.set_output_type(NEXATOM_OUTPUT_REALTIME_DATA)`. This instructs the FPGA data shuffler to actively route the accumulated realtime packets over the USB bulk endpoint to the host.
5.  **Acquisition Loop:** The main thread sleeps for the requested duration. The SDK's background threads continuously poll the USB interface and fire the registered Python callbacks.

> **Thread Safety.** Callbacks execute on a native background worker thread. While the SDK's C API is fully thread-safe, users must ensure their Python callback implementations (e.g., appending to lists, updating UI frameworks) incorporate appropriate threading locks or thread-safe queues.

### Execution

To run the realtime streaming tutorial:

```powershell
python python\examples\ftdi_realtime_smoke.py --duration-sec 5.0
```

**Expected Output:**

```text
Discovering NexatomTT devices.
Selected device: name=UTT810, serial=NTT-00000001, connection=FTDI:1.
Runtime firmware ready; enabling realtime output.
Telemetry seq=1 uptime_s=33 temp_c=42.10 mode=0x00 status=0x00
CPS period_ms=1000 total=0 channels=8 counts=[0, 0, 0, 0, 0, 0, 0, 0]
Telemetry seq=2 uptime_s=34 temp_c=42.15 mode=0x00 status=0x00
CPS period_ms=1000 total=0 channels=8 counts=[0, 0, 0, 0, 0, 0, 0, 0]
...
Observed callbacks: cps=5 telemetry=5
FTDI realtime smoke complete.
```
