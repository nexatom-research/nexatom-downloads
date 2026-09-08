# Quick Start Workflow

This section presents three end-to-end workflows that exercise the SDK from DLL load through hardware data acquisition and cleanup. Each subsection corresponds to an example script that can be run from the SDK root without modification.

> **Prerequisite.** All workflows require a connected UTT810 with the [FTDI D3XX driver installed](1_1_installation.md#ftdi-d3xx-driver-installation) and a valid runtime firmware image already loaded in at least one flash slot. The `open_runtime_device()` context manager handles boot automatically.

### Device lifecycle

All interaction with the UTT810 follows an eight-phase sequence. The phases must be executed in order.

```
Discover → Create → Connect → Configure → Acquire → Stop → Disconnect → Destroy
```

#### Minimal Python example

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
    # 4. Connect (USB handshake, 5 s timeout)
    device.connect(timeout_ms=5000)

    # 5. Register callbacks before enabling output
    def on_cps(data: NexatomCpsData) -> None:
        counts = [int(data.counts[i]) for i in range(data.num_channels)]
        print(f"CPS total={data.total_count} channels={counts}")

    device.set_count_rate_callback(on_cps)

    # 6. Configure and start acquisition
    device.enable_system(True)
    device.set_output_type(NEXATOM_OUTPUT_REALTIME_DATA)

    # 7. Collect data
    time.sleep(5.0)

    # 8. Stop and clean up
    device.set_output_type(NEXATOM_OUTPUT_NO_OUTPUT)
    device.enable_system(False)
    device.disconnect()
finally:
    device.destroy()
```

#### C API equivalent

```c
#include "nexatomtt_c_api.h"

nexatom_tt_info_t devices[8];
size_t count;
nexatom_tt_discover_devices(devices, 8, &count);

nexatom_tt_handle device;
nexatom_tt_create(&devices[0], &device);
nexatom_tt_connect(device, 5000);

nexatom_tt_enable_system(device, true);
nexatom_tt_set_output_type(device, NEXATOM_OUTPUT_REALTIME_DATA);

/* ... acquire data via callbacks ... */

nexatom_tt_set_output_type(device, NEXATOM_OUTPUT_NO_OUTPUT);
nexatom_tt_enable_system(device, false);
nexatom_tt_disconnect(device);
nexatom_tt_destroy(device);
```

#### Automated boot with `open_runtime_device()`

The UTT810 starts in a bootloader boot-decision window. The `open_runtime_device()` context manager automates the full boot sequence — slot table query, valid slot selection, boot command, USB re-enumeration, and runtime mode verification — then yields a connected, runtime-ready device handle:

```python
from nexatomtt import NexatomLibrary, open_runtime_device, RuntimeBootOptions

lib = NexatomLibrary(home=".")
devices = lib.discover_devices()

options = RuntimeBootOptions(
    timeout_ms=5000,          # Connect timeout
    mode_timeout_sec=20.0,    # Wait for runtime mode after boot
    poll_sec=0.25,            # Polling interval during boot
    preferred_slot=None,      # None = auto-select (default → lowest valid)
)

with open_runtime_device(lib, devices[0], options=options) as device:
    # device is connected and confirmed in RUNTIME mode
    device.enable_system(True)
    # ... configure and acquire ...
    device.enable_system(False)
# disconnect + destroy happen automatically on context exit
```

All example scripts in this chapter use `open_runtime_device()` for device initialization.

---

### Realtime CPS and telemetry streaming

**Script:** `python/examples/ftdi_realtime_smoke.py`

This smoke test verifies the end-to-end data path from FPGA to host. It registers count-rate (CPS) and telemetry callbacks, enables `REALTIME_DATA` output, streams for a configurable duration, and asserts that both callback types were invoked at least once.

#### Running the script

```powershell
python python\examples\ftdi_realtime_smoke.py --home . --duration-sec 10
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
5. enable_telemetry(True)               ← enable telemetry packets
6. set_telemetry_mode(2)                ← periodic telemetry
7. set_cps_period_selector(0)           ← 1000 ms CPS window
8. set_output_type(REALTIME_DATA)       ← start FPGA → host data flow
9. request_telemetry()                  ← request initial telemetry packet
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

The script exits with code `0` on success. Exit code `2` indicates that CPS or telemetry callbacks were never received — check the USB connection, FTDI driver, and firmware slot state.

---

### Raw time-tag capture and CSV export

**Script:** `python/examples/file_save_and_offline_decode.py`

This workflow captures individual time-stamped photon events to `.nxtt` binary files using `RAW_TAGS` output mode, then optionally decodes them offline to CSV.

#### Running the script

```powershell
python python\examples\file_save_and_offline_decode.py ^
    --home . --output-dir .\captures --duration-sec 10 --decode-csv
```

| Flag | Default | Description |
|---|---|---|
| `--home` | Auto-detected | SDK root containing `nexatomTT.dll` |
| `--output-dir` | `captures` | Directory for `.nxtt` output files |
| `--base-name` | `nexatomtt_capture` | Filename prefix for saved files |
| `--duration-sec` | `10.0` | Capture duration (s) |
| `--decode-csv` | Off | Decode newest `.nxtt` to CSV after capture |
| `--timeout-ms` | `5000` | Device connect timeout (ms) |
| `--boot-slot` | Auto | Explicit VALID slot index to boot |

#### Capture workflow

```
1. open_runtime_device()                         ← boot and connect
2. enable_system(True)                           ← start data shuffler
3. set_time_tag_file_config(config)              ← configure file saving
4. set_output_type(RAW_TAGS)                     ← start raw tag stream
5. enable_time_tag_file_saving(prefix, BINARY)   ← begin writing .nxtt
6. sleep(duration_sec)                           ← capture window
7. set_output_type(NO_OUTPUT)                    ← stop tag stream FIRST
8. disable_time_tag_file_saving()                ← close file SECOND
9. enable_system(False)                          ← disable data shuffler
```

> **Shutdown ordering.** Output mode must be set to `NO_OUTPUT` *before* disabling file saving. This prevents the native file writer from receiving new data while it is flushing and closing the output file.

#### File saving configuration

The script constructs a `NexatomTimeTagFileConfig` with these defaults:

```python
config = NexatomTimeTagFileConfig()
config.format             = NEXATOM_TT_FILE_BINARY   # .nxtt binary format
config.max_file_size_mb   = 256                       # Rotate at 256 MB
config.max_duration_minutes = ceil(duration_sec / 60) # Rotate on time
config.max_event_count    = 50_000_000                # Rotate on event count
config.rotate_on_acquisition_boundary = False
config.include_timestamp  = True                      # Timestamp in filename
config.flush_immediately  = False                     # Buffered writes
```

When any rotation limit is reached, the native library closes the current file and opens a new one with an incremented sequence number in the filename.

#### Offline CSV decode

With `--decode-csv`, the script opens the newest `.nxtt` file using the native binary reader and exports every tag as a `timestamp_ps,channel` CSV row:

```python
with library.open_time_tag_file_reader(nxtt_path) as reader:
    row_count = reader.export_csv(csv_path)
print(f"Exported {row_count} tags to {csv_path}")
```

Each row in the output CSV represents one decoded `nexatom_time_tag_t`:

| Column | Type | Description |
|---|---|---|
| `timestamp_ps` | `uint64` | Absolute event timestamp in picoseconds |
| `channel` | `uint8` | Input channel index (0–7) |

---

### Live TIHI and MFCO plotting

**Script:** `python/examples/tihi_mfco_matplotlib.py`

This script demonstrates the full realtime acquisition pipeline: Time Interval Histogram (TIHI) and Multifold Coincidence (MFCO) with live Matplotlib visualization. It is designed as an editable customer example — default constants at the top of the script can be modified for repeated lab use.

#### Running the script

First-run validation with internal test pulses (no external cables required):

```powershell
python -m pip install matplotlib
python python\examples\tihi_mfco_matplotlib.py
```

With external START/STOP signals:

```powershell
python python\examples\tihi_mfco_matplotlib.py --no-test-pulses --duration-sec 30
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
| `--test-pulse-period-cycles` | `125` | Period in 125 MHz clock cycles (~1 MHz) |
| `--test-pulse-width-cycles` | `10` | Pulse width in clock cycles |
| `--tihi-start-channel` | `0` | TIHI START channel |
| `--tihi-stop-channel` | `1` | TIHI STOP channel |
| `--tihi-stop-delay-ps` | `2000` | Input delay on STOP channel (ps) |
| `--tihi-bin-width-ps` | `1000` | Histogram bin width (ps) |
| `--tihi-bins` | `1023` | Number of histogram bins |
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
 1. reset_peripherals(True) → sleep(10ms) → reset_peripherals(False)
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
    set_time_histogram_num_bins(1023)
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

Optional `--save-plots` writes PNG files and `--save-csv` writes `tihi_histogram.csv` and `mfco_patterns.csv` to the output directory after acquisition completes.
