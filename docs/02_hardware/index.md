# Hardware

This chapter documents the hardware interface of the UTT810 time tagger as exposed through the SDK. All hardware behaviour described here is mediated by the FTDI D3XX USB transport and the native library — there is no direct register-level access from user code.

| Property | Value |
|---|---|
| Hardware model | UTT810 |
| Input channels | 8 (0–7) |
| Transport | FTDI FT60x USB 3.0 |
| Protocol modes | Runtime, Bootloader |
| Firmware storage | Flash slots (bootloader-managed) |

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
| [System Control](2_8_system_control.md)               | System enable, peripheral reset, sync clock, global stop |
| [Counts per Second (CPS)](2_9_cps_configuration.md)   | CPS measurement and period selector |
| [Telemetry](2_10_telemetry.md)                        | Device health, temperature, status monitoring |
| [Bootloader Handoff](211-bootloader-handoff)          | Runtime ↔ bootloader service entry |
| [External Clock Input](2_12_external_clock_input.md)  | Sync clock request |