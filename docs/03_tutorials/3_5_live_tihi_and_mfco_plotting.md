## Live TIHI and MFCO Plotting

This tutorial demonstrates how to configure the Time Interval Histogram (TIHI) and Multi-Fold Coincidence (MFCO) hardware engines for advanced analysis. It also provides a foundational example of processing the resulting asynchronous data streams into a live Matplotlib visualization.

**Relevant script:**
*   `tihi_mfco_matplotlib.py`

### Workflow

1.  **Boot & System Suspend:** The device is booted into runtime mode. As per best practices, the primary data shuffler is suspended (`device.enable_system(False)`) while the hardware acquisition engines are configured.
2.  **Configure TIHI (Time Interval Histogram):**
    *   **Routing:** `device.set_time_histogram_channels(start, stop)` assigns the physical channels acting as the START and STOP triggers.
    *   **Time Axis:** `device.set_time_histogram_bin_width(ps)` and `device.set_time_histogram_num_bins(count)` define bin width and span. Bin width is a histogram setting, not an analog timing-accuracy specification; validate the bin count against the native profile/capabilities.
    *   **Completion:** `device.set_time_histogram_stop_conditions(count, duration_ms, use_duration)` selects an event/count or duration completion condition. Do not infer automatic periodic rearming solely from setting a duration.
    *   **Enable:** `device.enable_time_histogram(True)` powers the submodule logic.
3.  **Configure MFCO (Multi-Fold Coincidence):**
    *   **Timing:** `device.set_multifold_coincidence_window(ps)` establishes the maximum allowable temporal drift between events to be considered coincident.
    *   **Patterns:** Current eight-input MFCO returns 256 pattern bins. The plotter's `--mfco-analysis-channels` selects a software analysis filter; it does not suppress other physical events in the FPGA.
    *   **Enable:** `device.enable_multifold_coincidence(True)` powers the submodule logic.
4.  **Data Path Enable & Start:** The FPGA data shuffler is re-enabled (`device.enable_system(True)`) to allow USB traffic. The host then explicitly triggers the histogram engines to begin accumulating tags via `device.start_time_histogram()` and `device.start_multifold_coincidence()`.
5.  **Live Plotting:** As the hardware triggers its stop/emit conditions, it pushes payload packets over USB. The registered Python callbacks receive `NexatomTihiData` and `NexatomMfcoData` objects in the background. The main Python thread consumes these cached snapshots and continuously renders a live Matplotlib visualization until the duration expires.

### Execution

To run the live plotting demonstration. By default, the script injects synthetic test pulses to ensure the plots are visible even if physical signal sources are not attached.

*(Note: This example requires the `matplotlib` package installed in the active Python environment).*

```powershell
python -m pip install matplotlib
python python/examples/tihi_mfco_matplotlib.py --home . --duration-sec 10 --save-plots
```

**Illustrative output** (paths and selected device depend on the run):

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

### Adapt the display to your experiment

Use `--no-test-pulses` with external START/STOP signals. The plotter's cycle defaults use the connected profile's pulse clock, so calculate period as `period_cycles / test_pulse_clock_hz`. To survey roughly 10 us-separated signals, use a histogram span that covers the interval before narrowing it. The first-stop mode and relative input delay affect which peak appears.

`--no-live-window --save-plots --save-csv` produces a headless plot and final snapshot CSVs. It does not continuously record every processed packet. For that use the [processed acquisition template](../01_getting_started/1_2_quick_start.md#configure-channels-and-record-processed-results), whose native sinks remain open throughout the measurement. Keep aggregation and completion metadata with the plotted data; a nonzero bar alone does not establish a completed, valid measurement.
