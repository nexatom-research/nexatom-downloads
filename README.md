# Nexatom Downloads & SDK Documentation

Welcome to the official repository for [Nexatom Research & Instruments](https://www.nexatom.in). This repository hosts the public releases, SDK binaries, and documentation for our precision scientific instruments, including the **UTT810 Universal Time Tagger**.

## About Nexatom

Nexatom mobilizes research knowledge into practical systems for precision light, timing, and scientific measurement. We develop indigenous high-technology solutions for precision lasers, time tagging, and custom scientific instrumentation used by research groups and industry teams.

## The NexatomTT SDK (UTT810)

The NexatomTT SDK provides host-side control of the UTT810 time tagger over USB 3.0. It features a native C-library and a Python wrapper for seamless integration into custom data pipelines, automated experiments, and live analysis (such as TIHI and MFCO plotting).

## [Documentation](https://nexatom-research.github.io/nexatom-downloads)

* **[Getting Started](docs/01_getting_started/index.md)**
    * [Installation Guide](docs/01_getting_started/1_1_installation.md)
        - [*System requirements*](docs/01_getting_started/1_1_installation.md#system-requirements)
        - [*FTDI D3XX driver installation*](docs/01_getting_started/1_1_installation.md#ftdi-d3xx-driver-installation)
        - [*SDK package contents*](docs/01_getting_started/1_1_installation.md#sdk-package-contents)
        - [*Verifying the installation*](docs/01_getting_started/1_1_installation.md#verifying-the-installation)
        - [*SDK home discovery*](docs/01_getting_started/1_1_installation.md#sdk-home-discovery)
    * [Quick Start Workflow](docs/01_getting_started/1_2_quick_start.md)
        - [*Device lifecycle*](docs/01_getting_started/1_2_quick_start.md#device-lifecycle)
        - [*Realtime CPS and telemetry streaming*](docs/01_getting_started/1_2_quick_start.md#realtime-cps-and-telemetry-streaming)
        - [*Raw time-tag capture and CSV export*](docs/01_getting_started/1_2_quick_start.md#raw-time-tag-capture-and-csv-export)
        - [*Live TIHI and MFCO plotting*](docs/01_getting_started/1_2_quick_start.md#live-tihi-and-mfco-plotting)
    * [Programming Languages](docs/01_getting_started/1_3_programming.md)
        - [*C / C++ integration*](docs/01_getting_started/1_3_programming.md#c-cpp-integration)
        - [*Python integration*](docs/01_getting_started/1_3_programming.md#python-integration)
        - [*FFI / Foreign language bindings*](docs/01_getting_started/1_3_programming.md#ffi-foreign-language-bindings)
    * [Firmware Management](docs/01_getting_started/1_4_firmware.md)
        - [*Firmware image format*](docs/01_getting_started/1_4_firmware.md#firmware-image-format)
        - [*Firmware manifest*](docs/01_getting_started/1_4_firmware.md#firmware-manifest)
        - [*Field update workflow*](docs/01_getting_started/1_4_firmware.md#field-update-workflow)
        - [*Loading a firmware image*](docs/01_getting_started/1_4_firmware.md#loading-a-firmware-image)
        - [*Slot management*](docs/01_getting_started/1_4_firmware.md#slot-management)
        - [*Safety precautions*](docs/01_getting_started/1_4_firmware.md#safety-precautions)
* **[Hardware](docs/02_hardware/index.md)**
    * [Operating Conditions](docs/02_hardware/2_1_operating_conditions.md)
        - [*USB 3.0 data connection*](docs/02_hardware/2_1_operating_conditions.md#usb-data-connection)
        - [*Device discovery and enumeration*](docs/02_hardware/2_1_operating_conditions.md#device-discovery-and-enumeration)
        - [*Connection management and state machine*](docs/02_hardware/2_1_operating_conditions.md#connection-management-and-state-machine)
        - [*Hardware protocol modes*](docs/02_hardware/2_1_operating_conditions.md#hardware-protocol-modes)
    * [Input Channels](docs/02_hardware/2_2_input_channels.md)
        - [*Channel count and indexing*](docs/02_hardware/2_2_input_channels.md#channel-count-and-indexing)
        - [*Threshold configuration*](docs/02_hardware/2_2_input_channels.md#threshold-configuration)
        - [*Edge type selection*](docs/02_hardware/2_2_input_channels.md#edge-type-selection)
        - [*Input hysteresis*](docs/02_hardware/2_2_input_channels.md#input-hysteresis)
        - [*Channel routing*](docs/02_hardware/2_2_input_channels.md#channel-routing)
    * [Data Connection](docs/02_hardware/2_3_data_connection.md)
        - [*FTDI USB 3.0 transport layer*](docs/02_hardware/2_3_data_connection.md#ftdi-usb-3-transport-layer)
        - [*Output data type modes*](docs/02_hardware/2_3_data_connection.md#output-data-type-modes)
        - [*Performance monitoring*](docs/02_hardware/2_3_data_connection.md#performance-monitoring)
    * [Calibration](docs/02_hardware/2_4_calibration.md)
        - [*Manual calibration trigger*](docs/02_hardware/2_4_calibration.md#manual-calibration-trigger)
        - [*Auto-calibration configuration*](docs/02_hardware/2_4_calibration.md#auto-calibration-configuration)
        - [*Calibration status via telemetry*](docs/02_hardware/2_4_calibration.md#calibration-status-via-telemetry)
        - [*Calibration data structure*](docs/02_hardware/2_4_calibration.md#calibration-data-structure)
    * [LEDs](docs/02_hardware/2_5_leds.md)
        - [*Per-channel LED control*](docs/02_hardware/2_5_leds.md#per-channel-led-control)
        - [*RGB LED mode selection*](docs/02_hardware/2_5_leds.md#rgb-led-mode-selection)
    * [Test Signal](docs/02_hardware/2_6_test_signal.md)
        - [*Enabling the internal test pulse generator*](docs/02_hardware/2_6_test_signal.md#enabling-the-internal-test-pulse-generator)
        - [*Configuring test pulse parameters*](docs/02_hardware/2_6_test_signal.md#configuring-test-pulse-parameters)
    * [Synthetic Input Delay](docs/02_hardware/2_7_synthetic_input_delay.md)
        - [*Per-channel input delay configuration*](docs/02_hardware/2_7_synthetic_input_delay.md#per-channel-input-delay-configuration)
    * [External Clock Input](docs/02_hardware/2_8_external_clock.md)
        - [*Requesting external sync-clock operation*](docs/02_hardware/2_8_external_clock.md#requesting-external-sync-clock-operation)
        - [*Sync-clock status monitoring via telemetry*](docs/02_hardware/2_8_external_clock.md#sync-clock-status-monitoring-via-telemetry)
    * [System Control](docs/02_hardware/2_9_system_control.md)
        - [*System enable / disable (Data Shuffler)*](docs/02_hardware/2_9_system_control.md#system-enable-disable)
        - [*Peripheral reset*](docs/02_hardware/2_9_system_control.md#peripheral-reset)
        - [*Global acquisition stop*](docs/02_hardware/2_9_system_control.md#global-acquisition-stop)
    * [Counts per Second (CPS)](docs/02_hardware/2_10_cps_configuration.md)
        - [*CPS integration window selection*](docs/02_hardware/2_10_cps_configuration.md#cps-integration-window-selection)
        - [*CPS data structure*](docs/02_hardware/2_10_cps_configuration.md#cps-data-structure)
    * [Telemetry](docs/02_hardware/2_11_telemetry.md)
        - [*Telemetry modes*](docs/02_hardware/2_11_telemetry.md#telemetry-modes)
        - [*Requesting telemetry on-demand*](docs/02_hardware/2_11_telemetry.md#requesting-telemetry-on-demand)
        - [*Polling latest telemetry*](docs/02_hardware/2_11_telemetry.md#polling-latest-telemetry)
        - [*Telemetry data fields*](docs/02_hardware/2_11_telemetry.md#telemetry-data-fields)
        - [*Configuration register dump*](docs/02_hardware/2_11_telemetry.md#configuration-register-dump)
    * [Runtime / Bootloader Handoff](docs/02_hardware/2_12_runtime_handoff)
        - [*Service entry for field upgrades*](docs/02_hardware/2_12_runtime_handoff.md#service-entry-for-field-upgrades)
        - [*Bootloader-first boot orchestration*](docs/02_hardware/2_12_runtime_handoff.md#bootloader-first-boot-orchestration)
        - [*Post-boot USB re-enumeration and device identity*](docs/02_hardware/2_12_runtime_handoff.md#post-boot-usb-re-enumeration-and-device-identity)
* **[Tutorials](docs/03_tutorials/index.md)**
    * [DLL Load and Version Check](docs/03_tutorials/3_1_dll_load_and_version_check.md)
    * [Booting into Runtime from Bootloader](docs/03_tutorials/3_2_booting_into_runtime_from_bootloader.md)
    * [Realtime CPS and Telemetry Streaming](docs/03_tutorials/3_3_realtime_cps_and_telemetry_streaming.md)
    * [Raw Time-Tag Capture and Offline CSV Export](docs/03_tutorials/3_4_raw_time_tag_capture_and_offline_csv_export.md)
    * [Live TIHI and MFCO Plotting](docs/03_tutorials/3_5_live_tihi_and_mfco_plotting.md)
    * [Runtime ↔ Bootloader Handoff Validation](docs/03_tutorials/3_6_runtime_bootloader_handoff_validation.md)
    * [End-to-End Firmware Field Update](docs/03_tutorials/3_7_end_to_end_firmware_field_update.md)
* **[Software Overview](docs/04_software_overview/index.md)**
    * [Architecture and Data Flow](docs/04_software_overview/index.md#architecture-and-data-flow)
        - [*Native library*](docs/04_software_overview/index.md#native-library)
        - [*Data pipeline*](docs/04_software_overview/index.md#data-pipeline)
        - [*Thread safety model*](docs/04_software_overview/index.md#thread-safety-model)
        - [*Memory management*](docs/04_software_overview/index.md#memory-management)
    * [Precompiled Libraries and Language Bindings](docs/04_software_overview/index.md#precompiled-libraries-and-language-bindings)
        - [*Native DLL and runtime dependencies*](docs/04_software_overview/index.md#native-dll-and-runtime-dependencies)
        - [*Python package (`nexatomtt`)*](docs/04_software_overview/index.md#python-package-nexatomtt)
    * [C API](docs/04_software_overview/index.md#c-api)
        - [*Header organization*](docs/04_software_overview/index.md#header-organization)
        - [*Error handling convention*](docs/04_software_overview/index.md#error-handling-convention)
        - [*Opaque handle pattern*](docs/04_software_overview/index.md#opaque-handle-pattern)
* **[Application Programming Interface](docs/05_api_reference/index.md)**

## Downloads

You can find the latest releases and binaries in the [Releases](https://github.com/nexatom-research/nexatom-downloads/releases) section.

## Support

If you encounter a bug, need to report an issue with the SDK, or have questions about integrating the API, please open an **[Issue](https://github.com/nexatom-research/nexatom-downloads/issues)** on this repository.

For custom instrumentation discussions or hardware support, please [contact us directly](https://www.nexatom.in/).

## License

This SDK and its documentation are provided under the Nexatom End-User License Agreement. See the `LICENSE.txt` file included in your release download for full terms. Third-party components (such as FTDI and MinGW runtimes) remain under their respective licenses.

---

**Last Updated:** Sep 10, 2026
