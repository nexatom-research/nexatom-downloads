## Live TIHI and MFCO Plotting

This tutorial demonstrates how to configure the Time Interval Histogram (TIHI) and Multi-Fold Coincidence (MFCO) hardware engines for advanced analysis. It also provides a foundational example of processing the resulting asynchronous data streams into a live Matplotlib visualization.

**Relevant script:**
*   `tihi_mfco_matplotlib.py`

### Workflow

1.  **Boot & System Suspend:** The device is booted into runtime mode. As per best practices, the primary data shuffler is suspended (`device.enable_system(False)`) while the hardware acquisition engines are configured.
2.  **Configure TIHI (Time Interval Histogram):**
    *   **Routing:** `device.set_time_histogram_channels(start, stop)` assigns the physical channels acting as the START and STOP triggers.
    *   **Resolution:** `device.set_time_histogram_bin_width(ps)` and `device.set_time_histogram_num_bins(count)` define the time-axis resolution.
    *   **Emissions:** `device.set_time_histogram_stop_conditions(count, duration_ms, use_duration)` determines how often the hardware emits a completed histogram packet (e.g., every 100 ms).
    *   **Enable:** `device.enable_time_histogram(True)` powers the submodule logic.
3.  **Configure MFCO (Multi-Fold Coincidence):**
    *   **Timing:** `device.set_multifold_coincidence_window(ps)` establishes the maximum allowable temporal drift between events to be considered coincident.
    *   **Routing:** `device.set_multifold_coincidence_channels(channels_list)` assigns the hardware channels participating in the coincidence matrix.
    *   **Enable:** `device.enable_multifold_coincidence(True)` powers the submodule logic.
4.  **Data Path Enable & Start:** The FPGA data shuffler is re-enabled (`device.enable_system(True)`) to allow USB traffic. The host then explicitly triggers the histogram engines to begin accumulating tags via `device.start_time_histogram()` and `device.start_multifold_coincidence()`.
5.  **Live Plotting:** As the hardware triggers its stop/emit conditions, it pushes payload packets over USB. The registered Python callbacks receive `NexatomTihiData` and `NexatomMfcoData` objects in the background. The main Python thread consumes these cached snapshots and continuously renders a live Matplotlib visualization until the duration expires.

### Execution

To run the live plotting demonstration. By default, the script injects synthetic test pulses to ensure the plots are visible even if physical signal sources are not attached.

*(Note: This example requires the `matplotlib` package installed in the active Python environment).*

```powershell
python python\examples\tihi_mfco_matplotlib.py --duration-sec 10.0
```

**Expected Output:**

```text
Discovering NexatomTT devices.
Selected device: name=UTT810, serial=NTT-00000001, connection=FTDI:1.
Runtime firmware ready; suspending system for configuration.
...
Configuring TIHI while acquisition is stopped.
Configuring MFCO while acquisition is stopped.
Starting TIHI and MFCO acquisition.
Live plotting started. Close the Matplotlib window or press Ctrl+C to abort.
Acquisition complete.
Saved plot captures to captures/tihi_histogram.png and captures/mfco_patterns.png.
```

*A Matplotlib graphical window will automatically spawn, updating at high frequency to display the realtime TIHI histogram distribution and the most active MFCO pattern bars.*
