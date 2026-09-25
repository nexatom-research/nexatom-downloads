# NexatomTT SDK manual

This manual explains how to connect a Nexatom time tagger, configure its inputs,
acquire measurements and work with the saved data from C, C++ or Python.
It follows the complete workflow: installation, your first recording, instrument
controls, worked examples, and the detailed API reference.

If you are using the SDK for the first time, begin with the installation guide
and quick start. If you are adapting an existing application, use the language
integration chapter together with the reference for the function you are changing.

| Chapter | What you will learn |
| --- | --- |
| [1. Getting started](01_getting_started/index.md) | Install on Windows/Linux, make a first measurement, integrate C/C++/Python and understand firmware slots |
| [2. Device operation](02_hardware/index.md) | Configure inputs, delays, test signals, clocks, calibration, rates and telemetry |
| [3. Tutorials](03_tutorials/index.md) | Follow the example programs through acquisition, raw event export, plotting and service operations |
| [4. Software overview](04_software_overview/index.md) | Understand the data path, device ownership and error handling |
| [5. Python reference](05_api_reference/index.md) | Use library/device classes, callback records and measurement controls |
| [6. In-depth guides](06_in_depth_guides/index.md) | Work with file formats, offline decoding, logging, scientific analysis and callback lifetimes |
| [7. C API reference](07_c_api/index.md) | Look up declarations, parameters and commented examples by function family |
| [8. Safety and licensing](08_safety_compliance/index.md) | Review operating precautions and package notices |

## Choose your first example

- **Configure inputs and record processed measurements:** follow the
  [processed quick start](01_getting_started/1_2_quick_start.md#configure-channels-and-record-processed-results).
  It saves count rates, time-interval histograms and coincidence results.
- **Record individual event times:** follow
  [raw capture and CSV export](03_tutorials/3_4_raw_time_tag_capture_and_offline_csv_export.md).
  Each decoded event has a timestamp in picoseconds and a channel.
- **Build a C or C++ application:** use the
  [standalone example project](01_getting_started/1_3_programming.md#c-cpp-examples).
- **View live histograms:** follow the
  [TIHI/MFCO plotting tutorial](03_tutorials/3_5_live_tihi_and_mfco_plotting.md).

Each example opens one board, selected by the USB port it is plugged into. An
application can open several boards at once, one handle each. The native API
handles automatic model discovery and runtime startup. If a bootloader-equipped device
already contains a valid runtime image, ordinary acquisition can boot it without
loading firmware. Original Zynq without a bootloader connects directly to runtime.

## Version scope and migration

This edition describes **SDK 0.1.0-preview.18.2**. Keep the complete extracted package,
including its native library, Python modules, header, dependencies and notices.
The native library's version string and the SDK archive version are separate
identifiers; use the package manifest to identify the installed SDK.

Changes since the preview.8 edition:

- **Board identity.** A board is selected by its USB port (`connection_id`,
  `usb:` plus the port path). The FT601 USB serial is information only.
  Several boards can be open at once.
- **Result model.** TIHI, MFCO, the correlators and Fast TIHI publish results
  with a `result_status` and a `result_span`. `set_result_span()`,
  `set_run_end()` and `clear_result()` replace the aggregation modes and
  stop conditions.
- **Connection.** `connect()` returns once the device can be controlled.
  Automatic reconnection is an explicit opt-in (`set_auto_reconnect`).
- **File saving.** `nexatom_tt_get_file_saving_statistics()` proves that a
  capture is complete.
- **Callbacks.** Callback data, including Fast TIHI bins, is passed by value.
- **Firmware.** Images are named `Z080_nnn` (UTT810) and `K168_nnn`
  (UTT160810) and published in firmware catalogues.

The chapter structure, explanations and worked examples are retained. Examples distinguish complete programs from fragments and explain
where to change settings. Device limits come from the active native profile and
the specification for your instrument; a numerical API unit alone is not a
physical resolution or accuracy rating.

See the [release page](https://www.nexatom.in/downloads/utt810/sdk-preview/) for
packages and platform information. Firmware is supplied separately when an
intentional image update is needed. The
[file-format guide](06_in_depth_guides/6_1_file_saving_and_data_export.md) describes
the processed CSV/TAB and HDF5 layouts written by this release and by earlier ones.
