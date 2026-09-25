# Hardware

This chapter explains the UTT810 hardware interface as exposed through SDK 0.1.0-preview.18.2: how to connect an instrument, configure its inputs, choose an output path, and interpret hardware status. The native library owns the FTDI transport, protocol selection and hardware commands. Examples use physical units and the resolved device profile rather than raw register writes.

| Property | Value |
|---|---|
| Supported workflow | One handle per board; several boards can be open at once, each selected by its USB port |
| Hardware families | Original Zynq runtime, bootloader-equipped Zynq, and bootloader-equipped Kintex through one native API |
| Public input channels | Up to 8 (0–7), restricted by the effective public channel mask |
| Transport | FTDI FT60x USB 3.0 |
| Protocol modes | Runtime; service/bootloader where supported |
| Firmware storage | Bootloader-managed slots on equipped instruments |

The firmware advertises identity/layout/features, and the API resolves the supported settings and units for that model/image. Input count, digital output count, input-delay limit and test-pulse clock are distinct properties. Query them rather than assuming all generations are interchangeable. Accepted configuration ranges are not electrical safety ratings; use the supplied instrument specifications for external signal amplitude, termination and operating environment.

## Chapter contents

| Section                                               | Topic |
|-------------------------------------------------------|---|
| [Operating Conditions](2_1_operating_conditions.md)   | USB connection, device discovery, state machine, protocol modes |
| [Input Channels](2_2_input_channels.md)               | Threshold, edge type, hysteresis, routing, test pulses |
| [Data Connection](2_3_data_connection.md)             | Output modes, CPS period, data flow architecture |
| [Calibration](2_4_calibration.md)                     | Calibration data and workflow |
| [LEDs](2_5_leds.md)                                   | Channel LEDs and RGB LED control |
| [Test Signal](2_6_test_signal.md)                     | Internal test pulse generator |
| [Synthetic Input Delay](2_7_synthetic_input_delay.md) | Per-channel input delay |
| [External Clock Input](2_8_external_clock.md)         | Capability check, reference request and lock/active verification |
| [System Control](2_9_system_control.md)               | System enable, peripheral reset and global stop |
| [Counts per Second (CPS)](2_10_cps_configuration.md)   | CPS measurement, normalized rates and period selector |
| [Telemetry](2_11_telemetry.md)                        | Device health, temperature, status validity and configuration dumps |
| [Bootloader Handoff](2_12_runtime_handoff.md)          | Native runtime entry and runtime ↔ service transitions |
