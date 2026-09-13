# 5 Application Programming Interface

This chapter serves as the definitive reference for the high-level Python application programming interface and its underlying data structures. While Chapter 7 provides an exhaustive, low-level dictionary of every C ABI function, this chapter maps those functions into object-oriented Python classes and details the exact memory layouts of the data payloads returned by the hardware.

Use this chapter to reference constants, error codes, Python class methods, and the specific fields available in data callbacks (such as TIHI fit parameters or MFCO pattern counts).

## Chapter contents

| Topic                                                                 | Description |
|-----------------------------------------------------------------------|---|
| [The NexatomTT Library](5_1_nexatomtt_library.md)                     | Global constants, limits, error codes (`nexatom_error_code_t`), and hardware state enums. |
| [NexatomTT Classes](5_2_nexatomtt_classes.md)                         | Reference for `NexatomLibrary`, `NexatomDevice`, `NexatomTimeTagReader`, and runtime orchestrators. |
| [Data Structures](5_3_data_structures.md)                             | Field-by-field definitions for telemetry, CPS, TIHI, MFCO, Correlation, and firmware status payloads. |
| [Callback Types](5_4_callback_types.md)                               | Function signatures for asynchronous data dispatch and event notifications. |
| [Measurement Modules](5_5_measurement_modules.md)                     | A categorized overview of the functional API groups (TIHI, MFCO, DLS, FCS, DCS). |
| [MFCO Pattern Analysis Helpers](5_6_mfco_pattern_analysis_helpers.md) | Reference for the `nexatomtt.analysis` module (pattern masking, decoding, and counting). |
