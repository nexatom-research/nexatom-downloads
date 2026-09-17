# Quick Start Workflow

This section follows the instrument from library loading through channel configuration, acquisition, saving and cleanup. The examples cover live rates, processed histograms/coincidences, individual raw time tags and optional plotting. Run the supplied scripts from the extracted SDK root; use `python3` on Linux where appropriate.

> **Prerequisite.** Connect one intended instrument and complete [driver/device access setup](1_1_installation.md#ftdi-d3xx-driver-installation). Bootloader-equipped instruments need an existing usable runtime slot; original Zynq without a bootloader attaches directly to its runtime. The native API selects the supported startup path automatically. Close any other application using this instrument.

### [Device lifecycle](1_2_quick_start.md#device-lifecycle)

All interaction with the UTT810 follows an eight-phase sequence. The phases must be executed in order.

```
Discover → Create → Enter runtime → Configure → Acquire → Stop/finalize → Disconnect → Destroy
```

#### Minimal Python example

Save this as `python/first_cps.py` in the extracted SDK and run `python python/first_cps.py` from its root. It observes an external signal using existing channel settings. The processed example below demonstrates how to configure those settings explicitly and enable internal pulses.

```python
from nexatomtt import (
    NexatomLibrary,
    NexatomCpsData,
    NEXATOM_OUTPUT_REALTIME_DATA,
    NEXATOM_OUTPUT_NO_OUTPUT,
)
import time

# 1. Load native library
lib = NexatomLibrary(home=".")

# 2. Discover connected FTDI devices
devices = lib.discover_devices()
if not devices:
    raise RuntimeError("No NexatomTT devices found")

# 3. Create device handle
device = lib.create_device(devices[0])

try:
    # 4. Native attaches or boots an existing runtime on this same handle.
    device.connect_runtime(timeout_ms=20000)

    # 5. Register callbacks before enabling output
    def on_cps(data: NexatomCpsData) -> None:
        counts = [int(data.counts[i]) for i in range(data.num_channels)]
        print(f"CPS total={data.total_count} Hz channels={counts}")

    device.set_count_rate_callback(on_cps)

    # 6. Configure and start acquisition
    device.set_output_type(NEXATOM_OUTPUT_NO_OUTPUT)
    device.enable_system(False)
    device.set_cps_period_selector(0)  # One-second integration window.
    device.set_output_type(NEXATOM_OUTPUT_REALTIME_DATA)
    device.enable_system(True)

    # 7. Collect data
    time.sleep(5.0)

finally:
    # Try every cleanup operation, including after a configuration failure.
    # A completed stop must not prevent attempting disconnect and destruction.
    errors = []
    for action in (
        lambda: device.set_output_type(NEXATOM_OUTPUT_NO_OUTPUT),
        lambda: device.enable_system(False),
        device.disconnect,
    ):
        try:
            action()
        except Exception as exc:
            errors.append(str(exc))
    device.destroy()
    if errors:
        raise RuntimeError("Cleanup failed: " + "; ".join(errors))
```

#### C API equivalent

The corresponding calls are shown below as a fragment. Check every returned `nexatom_error_code_t`; [the complete minimal C example](1_3_programming.md#minimal-c-example) demonstrates error paths and cleanup.

```c
#include "nexatomtt_c_api.h"

nexatom_tt_info_t devices[8];
size_t count;
nexatom_tt_discover_devices(devices, 8, &count);

nexatom_tt_handle device;
nexatom_tt_create(&devices[0], &device);
nexatom_tt_connect_runtime(device, 20000);

nexatom_tt_enable_system(device, true);
nexatom_tt_set_output_type(device, NEXATOM_OUTPUT_REALTIME_DATA);

/* ... acquire data via callbacks ... */

nexatom_tt_set_output_type(device, NEXATOM_OUTPUT_NO_OUTPUT);
nexatom_tt_enable_system(device, false);
nexatom_tt_disconnect(device);
nexatom_tt_destroy(device);
```

#### Automated boot with `open_runtime_device()`

The `open_runtime_device()` context manager creates one device handle and calls native runtime entry. The API attaches to an already-running supported runtime or selects an existing valid slot when boot is needed. It establishes usable runtime state before yielding the handle. USB description strings, a boot acknowledgement or a fixed sleep are not substitutes for this operation.

```python
from nexatomtt import NexatomLibrary, open_runtime_device, RuntimeBootOptions

lib = NexatomLibrary(home=".")
devices = lib.discover_devices()

options = RuntimeBootOptions(
    timeout_ms=20000,         # Budget for native attach/boot/readiness
    preferred_slot=None,     # Normal automatic entry: valid default, then lowest valid
)

with open_runtime_device(lib, devices[0], options=options) as device:
    # device is connected and confirmed in RUNTIME mode
    profile = device.get_device_profile()
    print(f"Available inputs: 0x{profile.effective_public_tdc_mask:x}")
    # Configure and acquire using the workflows below.
    device.disconnect()  # Report disconnect errors explicitly.
# The context destroys the handle on both normal and exceptional exits.
```

The measurement scripts use this helper. It does not load firmware or change the persistent default slot. `mode_timeout_sec` and `poll_sec` remain options for the advanced explicit-slot workflow; normal startup uses the native `timeout_ms` budget. An explicit preferred slot is selected when entering from service; an already-running runtime is attached to rather than forcibly replaced.

---

### [Realtime CPS and telemetry streaming](1_2_quick_start.md#realtime-cps-and-telemetry-streaming)

**Script:** `python/examples/ftdi_realtime_smoke.py`

This smoke test verifies the end-to-end data path from FPGA to host. It registers count-rate (CPS) and telemetry callbacks, enables `REALTIME_DATA` output, streams for a configurable duration, and asserts that both callback types were invoked at least once.

#### Running the script

```powershell
python python/examples/ftdi_realtime_smoke.py --home . --timeout-ms 20000 --duration-sec 10
```

| Flag | Default | Description |
|---|---|---|
| `--home` | Auto-detected | SDK root containing `nexatomTT.dll` |
| `--timeout-ms` | `5000` | Device connect timeout (ms) |
| `--duration-sec` | `10.0` | Callback collection duration (s) |
| `--max-devices` | `8` | Maximum FTDI devices to discover |
| `--boot-slot` | Auto | Explicit VALID slot index to boot |
| `--mode-timeout-sec` | `20.0` | Runtime boot/reconnect timeout (s) |
| `--poll-sec` | `0.25` | Boot polling interval (s) |

#### Hardware setup sequence

The script executes the following call sequence after `open_runtime_device()` yields a connected device:

```
1. set_connection_status_callback()     ← monitor USB state
2. set_count_rate_callback()            ← register CPS handler
3. set_telemetry_callback()             ← register telemetry handler
4. enable_system(True)                  ← start data shuffler
5. enable_telemetry(True)               ← compatibility control when TELM is supported
6. set_telemetry_mode(2)                ← legacy periodic-mode request
7. set_cps_period_selector(0)           ← 1000 ms CPS window
8. set_output_type(REALTIME_DATA)       ← start FPGA → host data flow
9. request_telemetry()                  ← explicitly request a telemetry packet
10. sleep(duration_sec)                 ← collect callbacks
```

#### Expected output

```
Using first discovered NexatomTT device:
  serial_number:    NTT-XXXXXXXX
  firmware_version: X.X.X
  hardware_version: X.X
  device_name:      UTT810
  connection_type:  FTDI
  connection_id:    ...
Runtime firmware ready; enabling realtime output.
CPS period_ms=1000 total=... channels=8 counts=[...]
Telemetry seq=1 uptime_s=... temp_c=XX.XX mode=0x03 status=0x00
...
Observed callbacks: cps=N telemetry=N
FTDI realtime smoke complete.
```

The script exits with code `0` on success. Exit code `2` can indicate missing required callbacks, a refused profile/control request or a runtime/native error. Read the printed diagnostic before checking the USB connection, device access, signal source or firmware state.

The script only requires telemetry when the native profile advertises it. Current request-based firmware may return one telemetry packet for the explicit request; the legacy mode request is not a promise of a periodic stream. For a live telemetry display, request refreshes at the desired interval outside the callback. CPS `counts[]` and `total_count` already contain rates in Hz; do not multiply them by `1000 / measurement_period_ms` again.

---

### Configure channels and record processed results

**Script:** `python/examples/processed_acquisition.py`

This is the complete editable acquisition template. It configures threshold, edge, input delay and hysteresis, records native CPS/TIHI/MFCO CSV files, and calculates a small summary. For a first run without external signals:

```sh
python python/examples/processed_acquisition.py --home . --channels 0,1 --threshold-mv 500 --edge rising --delay-ps 0 --hysteresis-mv 10 --internal-test --duration-sec 5 --tihi-bin-width-ps 64000 --tihi-bins 256 --mfco-window-ps 12000000
```

| Setting | Meaning |
|---|---|
| `--channels 0,1` | First two selected public inputs are TIHI START/STOP; selected inputs also define the MFCO analysis mask |
| `--threshold-mv 500` | Comparator threshold requested for each selected input, in mV |
| `--edge rising` | Rising-edge detection; `falling` is also available |
| `--delay-ps 0` | Requested input delay in integer ps, checked against the native profile |
| `--hysteresis-mv 10` | Requested comparator hysteresis in mV |
| `--internal-test` | Generate pulses using timing derived from the profile, targeting about 100 kHz |
| `--tihi-bin-width-ps 64000` | 64 ns bins; 256 bins cover 16.384 us |
| `--mfco-window-ps 12000000` | 12 us coincidence window for a broad initial survey |

At 100 kHz, events within one channel are about 10 us apart. Relative channel phase can differ, so a narrow initial TIHI or coincidence window may show zero even when both inputs count. The broad survey above helps locate the distribution; narrow it for your experiment. A 12 us window can include successive pulses and is not a precision coincidence setting.

#### How the template is organized

1. Create a new run directory and enter runtime through the native API.
2. Read the profile and validate the complete channel plan before writing controls.
3. Quiet the system/output, stop inherited histogram runs and configure the selected inputs.
4. Configure TIHI/MFCO while stopped; register short callbacks and open native CSV sinks.
5. Enable output/system, start the measurement engines and enable test pulses if requested.
6. Retain callback snapshots while the native file writer records continuously.
7. Stop the engines, allow a bounded terminal-result wait, quiet the source, finalize files and check disconnect.
8. Reopen the CSV files and write `processed_summary.json` with data and completion/quality information.

Edit `ChannelSettings` in `channel_setup.py` for different values per input. For example, use 0 ps on START and 200000 ps on STOP to request a 200 ns relative delay, provided the profile permits it. The CLI's `--delay-ps` applies the same value to all selected inputs, so changing it alone does not introduce a relative START/STOP delay. Supported bootloader Zynq/Kintex profiles can expose a ceiling of 256000 ps; always read the active profile instead of assuming that limit for legacy hardware.

#### Reading the result

Each run creates `captures/processed-<id>/`. Native files contain one decoded result per record; `processed_summary.json` analyzes the latest snapshot. The template uses **REPLACE** aggregation, so it does not sum overlapping accumulated histograms. TIHI reports the bin sum and peak-bin start in ps. MFCO reports both exact selected-channel patterns and patterns containing those channels plus others. Completion reasons and error/quality flags remain beside the values.

Omit `--internal-test` for external signals. Internal pulses exercise digital acquisition and saving; evaluating threshold, hysteresis and physical edge response requires an external source. The same workflow is available in the [C/C++ example project](1_3_programming.md#c-cpp-examples).

---

### [Raw time-tag capture and CSV export](1_2_quick_start.md#raw-time-tag-capture-and-csv-export)

**Script:** `python/examples/file_save_and_offline_decode.py`

This workflow captures individual time-stamped photon events to `.nxtt` binary files using `RAW_TAGS` output mode, then optionally decodes them offline to CSV.

#### Running the script

```sh
python python/examples/file_save_and_offline_decode.py --home . --output-dir captures --duration-sec 5 --channels 0,1 --internal-test --decode-csv
```

| Flag | Default | Description |
|---|---|---|
| `--home` | Auto-detected | SDK root containing `nexatomTT.dll` |
| `--output-dir` | `captures` | Directory for `.nxtt` output files |
| `--duration-sec` | `5.0` | Capture duration (s) |
| `--decode-csv` | Off | Decode every new `.nxtt` file to CSV after capture |
| `--timeout-ms` | `20000` | Native runtime-entry budget (ms) |
| `--channels`, `--threshold-mv`, `--edge`, `--delay-ps`, `--hysteresis-mv` | As above | Shared channel configuration |
| `--internal-test` | Off | Use generated pulses; otherwise measure external inputs |

#### Capture workflow

```
1. open_runtime_device()                         ← native runtime entry
2. set_output_type(NO_OUTPUT), enable_system(False)
3. configure_channels(...), set_time_tag_file_config(config)
4. enable_time_tag_file_saving(prefix, BINARY)    ← prepare sink before events
5. set_output_type(RAW_TAGS), enable_system(True)
6. enable optional test pulses; observe for duration_sec
7. disable pulses; stop system/output
8. disable_time_tag_file_saving()                ← finalize before offline reading
9. disconnect()                                 ← check close result
```

> **Shutdown ordering.** Stop producers and quiet output before finalizing saving. Keep the device and file pipeline alive until those steps complete. Finalization must succeed before opening the captured file with an offline reader.

#### File saving configuration

The script constructs a `NexatomTimeTagFileConfig` with these defaults:

```python
config = NexatomTimeTagFileConfig()
config.format             = NEXATOM_TT_FILE_BINARY   # .nxtt binary format
config.max_file_size_mb   = 256                       # Rotate at 256 MB
config.max_duration_minutes = max(1, math.ceil(duration_sec / 60)) # Rotate on time
config.max_event_count    = 50_000_000                # Rotate on event count
config.rotate_on_acquisition_boundary = False
config.include_timestamp  = True                      # Timestamp in filename
config.flush_immediately  = False                     # Buffered writes
```

When any rotation limit is reached, the native library closes the current file and opens a new one with an incremented sequence number in the filename.

#### Offline CSV decode

With `--decode-csv`, the script opens every `.nxtt` file from the new run, including rotations, using the native binary reader and exports each individual tag as a `timestamp_ps,channel` CSV row:

```python
with library.open_time_tag_file_reader(nxtt_path) as reader:
    row_count = reader.export_csv(csv_path)
print(f"Exported {row_count} tags to {csv_path}")
```

Each row in the output CSV represents one decoded `nexatom_time_tag_t`:

| Column | Type | Description |
|---|---|---|
| `timestamp_ps` | `uint64` | Instrument event timestamp in picoseconds; not Unix wall-clock time |
| `channel` | `uint8` | Input channel index (0–7) |

The script also writes `raw_summary.json` with per-channel event counts and inter-arrival statistics. Subtract timestamps as integers before converting the difference to seconds. The Python summary excludes intervals across file boundaries and starts a new segment on a backwards timestamp; it records those boundaries rather than assuming clock continuity.

---

### [Live TIHI and MFCO plotting](1_2_quick_start.md#live-tihi-and-mfco-plotting)

**Script:** `python/examples/tihi_mfco_matplotlib.py`

This script demonstrates the full realtime acquisition pipeline: Time Interval Histogram (TIHI) and Multifold Coincidence (MFCO) with live Matplotlib visualization. It is designed as an editable customer example — default constants at the top of the script can be modified for repeated lab use.

#### Running the script

First-run validation with internal test pulses (no external cables required):

```powershell
python -m pip install matplotlib
python python/examples/tihi_mfco_matplotlib.py
```

With external START/STOP signals:

```powershell
python python/examples/tihi_mfco_matplotlib.py --no-test-pulses --duration-sec 30
```

Windows users can also launch via the bundled script:

```
python\examples\Launch TIHI MFCO Plotter.cmd
```

#### CLI arguments

| Flag | Default | Description |
|---|---|---|
| `--home` | Auto-detected | SDK root |
| `--duration-sec` | `10.0` | Capture duration (s) |
| `--use-test-pulses` | On | Enable internal test pulses for cable-free validation |
| `--no-test-pulses` | — | Use real external signals |
| `--test-pulse-channels` | `0,1` | Channels for test pulse generation |
| `--test-pulse-period-cycles` | `125` | Period in the profile's clock cycles; 1 MHz at 125 MHz, 2 MHz at 250 MHz |
| `--test-pulse-width-cycles` | `10` | Pulse width in clock cycles |
| `--tihi-start-channel` | `0` | TIHI START channel |
| `--tihi-stop-channel` | `1` | TIHI STOP channel |
| `--tihi-stop-delay-ps` | `2000` | Input delay on STOP channel (ps) |
| `--tihi-bin-width-ps` | `1000` | Histogram bin width (ps) |
| `--tihi-bins` | `1024` | Script default; select a lower count if the native capability ceiling requires it |
| `--tihi-stop-duration-ms` | `1000` | Hardware stop condition (ms) |
| `--mfco-window-ps` | `1000000` | Coincidence window (ps) |
| `--mfco-analysis-channels` | `0,1` | Software analysis/display filter channels |
| `--channel-threshold-mv` | `1000` | Threshold applied to channels 0–3 |
| `--cps-wait-sec` | `5.0` | CPS proof-of-life timeout (s) |
| `--save-plots` | Off | Save PNG plots to `--output-dir` |
| `--save-csv` | Off | Save CSV data to `--output-dir` |
| `--no-live-window` | Off | Headless mode (Agg backend) |

#### Acquisition lifecycle

The script follows a strict 10-step hardware sequence:

```
 1. Native runtime entry and profile validation
 2. Register CPS, TIHI, and MFCO callbacks
 3. Preflight stop/disable TIHI and MFCO (clear stale state)
 4. enable_system(True)
    enable_telemetry(True), set_telemetry_mode(2)
    set_cps_period_selector(0)
    set_output_type(REALTIME_DATA)
    set_channel_threshold(ch, 1000) for channels 0–3
 5. [Optional] Enable test pulses on configured channels:
    set_channel_test_pulse_params(ch, 125, 10)
    enable_channel_test_pulse(ch, True)
 6. CPS gate: wait up to 5 s for non-zero CPS on all required channels
 7. Configure TIHI (while processor is disabled):
    set_time_histogram_channels(start, stop)
    set_channel_input_delay(stop_ch, delay_ps)
    set_time_histogram_bin_width(1000)
    set_time_histogram_num_bins(selected_bins)  # Validated against native capabilities.
    set_time_histogram_stop_conditions(0, 1000, True)
    set_time_histogram_aggregation_mode(ACCUMULATE)
    enable_time_histogram(True)
 8. Configure MFCO (while processor is disabled):
    set_multifold_coincidence_window(1000000)
    set_multifold_coincidence_channels([0,1,2,3,4,5,6,7])
    set_multifold_coincidence_stop_conditions(0, 0, False)
    set_multifold_coincidence_aggregation_mode(ACCUMULATE)
    enable_multifold_coincidence(True)
 9. start_time_histogram() + start_multifold_coincidence()
    Live Matplotlib update loop for duration_sec
10. Cleanup (reverse order):
    stop TIHI → stop MFCO
    disable test pulses
    disable TIHI → disable MFCO
    set_output_type(NO_OUTPUT)
    enable_telemetry(False)
    enable_system(False)
```

> **CPS gate.** The script waits for non-zero per-channel CPS on every required measurement channel (TIHI start, TIHI stop, and MFCO analysis channels) before starting TIHI and MFCO processors. This prevents starting histogram acquisition before the timestamp stream is confirmed active. If CPS is not observed within the timeout, the script raises `CpsProofError` and exits with code `3`.

> **MFCO channel contract.** The 8-channel UTT810 hardware reports all 256 MFCO pattern bins regardless of the channel selector. The `--mfco-analysis-channels` flag controls only the software-side display and CSV analysis filter — it does not suppress events in the FPGA.

#### Live plot output

The Matplotlib window displays two panels refreshed every 250 ms:

- **Top panel** — Time Interval Histogram: bin counts (y) vs. time in nanoseconds (x), computed as `bin_index × bin_width_ps / 1000`.
- **Bottom panel** — MFCO Top Patterns: bar chart of the 10 highest-count coincidence patterns, labeled by channel combination (e.g., `0,1` for pattern 3).

Optional `--save-plots` writes PNG files and `--save-csv` writes `tihi_histogram.csv` and `mfco_patterns.csv` to the output directory after acquisition completes. These CSVs are the plotter's final snapshots. Use `processed_acquisition.py` for continuous native packet recording. The plotter's explicit cycle defaults can differ from the headless template's approximately 100 kHz pulse settings; read its reported clock and settings when interpreting timing.
