## Realtime CPS and Telemetry Streaming

Once the device is booted into the runtime firmware, the host can configure it to emit realtime diagnostics and event statistics. This tutorial demonstrates how to register Python callbacks and correctly sequence the hardware enable flags to receive a continuous stream of CPS (Counts Per Second) and telemetry packets.

**Relevant script:**
*   `ftdi_realtime_smoke.py`

### Workflow

1.  **Enter Runtime:** The script calls `open_runtime_device()` and checks the resolved profile (as detailed in Section 3.2). It registers telemetry only when the image advertises TELM.
2.  **Register Callbacks:** Python functions are defined and passed to the native SDK via `device.set_count_rate_callback()` and `device.set_telemetry_callback()`. When the native background thread receives and decodes a relevant packet, these Python functions are invoked asynchronously with `NexatomCpsData` and `NexatomTelemetryData` structures.
3.  **Enable Hardware Subsystems:** The hardware modules are powered up and configured:
    *   `device.enable_system(True)`: Activates the primary FPGA data shuffler.
    *   `device.enable_telemetry(True)` / `device.set_telemetry_mode(2)`: Requests the historical periodic telemetry controls. Current request-based firmware may not implement periodic emission through these controls.
    *   `device.set_cps_period_selector(0)`: Configures the CPS integration window to 1000 ms.
4.  **Route Output:** The final step calls `device.set_output_type(NEXATOM_OUTPUT_REALTIME_DATA)`. This instructs the FPGA data shuffler to actively route the accumulated realtime packets over the USB bulk endpoint to the host.
5.  **Acquisition Loop:** The script explicitly calls `request_telemetry()` when supported, then observes callbacks for the requested duration. The native transport receives and decodes data independently. At least one CPS callback and, when supported, one telemetry callback are required. A request-based image need not produce one telemetry packet per second.

> **Thread Safety.** Data callbacks normally arrive on native workers; some callback families can also run during API calls. Transfer owned records to a synchronized queue for UI work or slow processing. Do not destroy the device inside a callback. The [callback guide](../06_in_depth_guides/6_5_callback_thread_safety_and_data_lifetime.md) explains lifetime rules.

### Execution

To run the realtime streaming tutorial:

```powershell
python python/examples/ftdi_realtime_smoke.py --home . --timeout-ms 20000 --duration-sec 5.0
```

**Illustrative output:** callback counts and telemetry cadence depend on the image and requests; zero CPS values are legitimate without input events.

```text
Using first discovered NexatomTT device:
  serial_number:    000000000001
  firmware_version: X.X.X
  hardware_version: X.X
  device_name:      UTT810
  connection_type:  FTDI
  connection_id:    usb:PCIROOT(0)#PCI(0801)#PCI(0004)#USBROOT(0)#USB(4)
Runtime firmware ready; enabling realtime output.
Telemetry seq=1 uptime_s=33 temp_c=42.10 mode=0x00 status=0x00
CPS period_ms=1000 total=0 channels=8 counts=[0, 0, 0, 0, 0, 0, 0, 0]
Telemetry seq=2 uptime_s=34 temp_c=42.15 mode=0x00 status=0x00
CPS period_ms=1000 total=0 channels=8 counts=[0, 0, 0, 0, 0, 0, 0, 0]
...
Observed callbacks: cps=5 telemetry=5
FTDI realtime smoke complete.
```

For your own periodic display, keep request scheduling outside the callback:

```python
# Fragment inside an already-ready device session with its callback registered.
for _ in range(5):
    device.request_telemetry()
    time.sleep(1.0)  # Display refresh interval, not an initialization delay.
```

Use `get_telemetry_view()` for a cache-only view; it can return `None` before data arrives. The legacy `get_telemetry()` sends a request and waits for a result, so it is not interchangeable with the cached view. CPS rates are already in Hz despite their `counts` field names. This smoke script verifies callback delivery; use the processed template for configured test pulses and continuous saving.
