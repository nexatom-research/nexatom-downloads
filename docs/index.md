# NexatomTT SDK manual

The SDK provides the same native device API to C, C++ and Python applications on Windows and Linux. Begin with an extracted package and one connected device.

| Chapter | Use it for |
| --- | --- |
| [1. Getting started](01_getting_started/index.md) | Installation, first acquisition and language integration |
| [2. Device operation](02_hardware/index.md) | Identity, capabilities, inputs, output and telemetry |
| [3. Tutorials](03_tutorials/index.md) | Annotated processed/raw workflows and service tools |
| [4. Software overview](04_software_overview/index.md) | Ownership, errors and the data path |
| [5. Python reference](05_api_reference/index.md) | Classes, records, callbacks and analysis controls |
| [6. In-depth guides](06_in_depth_guides/index.md) | Files, logging, fitting and lifetime rules |
| [7. C API reference](07_c_api/index.md) | C/C++ declarations and feature families |
| [8. Safety and licensing](08_safety_compliance/index.md) | Package notices and firmware precautions |

## Version scope and migration

This edition describes **0.1.0-preview.7** and updates the preview.6 manual. The migration covers Windows/Linux installation, native runtime startup, device profiles, expanded Python controls, callback ownership and file formats.

The SDK release version and native `version()` string are separate identifiers. Retain the package's build information with measurement results. The shipped `include/nexatomtt_c_api.h` and Python modules are the exact declarations for that package; the manual explains how to use them.

The API behaviour described here matches published preview.7. Expanded inline teaching comments in the source templates accompany the next package update; this manual does not imply that previously published archives have been replaced or relabelled.

For existing applications:

1. Keep the full Windows or Linux package and use its loader and examples.
2. Replace client-managed boot/re-enumeration with `connect_runtime` or Python `open_runtime_device`; retain the same device handle.
3. Read the native profile before choosing channels, modes, delay or test-pulse parameters. A USB description or shortened telemetry serial does not establish the model.
4. Explicitly start the intended measurement. Connecting, selecting an output mode and enabling file saving are distinct actions.
5. Follow callback, result-validity and file-format rules; treat errors and empty results separately from successful acquisition.

The published preview.7 acquisition and lifecycle checks cover the documented Windows/Linux Zynq workflows, including internal test pulses and delay settings above 200 ns. They do not establish analog-input accuracy, maximum sustained throughput, inter-device synchronisation, or qualification of every firmware/model combination. Obtain electrical and environmental limits from the specification for your exact instrument; SDK units alone do not establish physical resolution or accuracy.

The examples are starting points for **one active device**. Multiple devices attached to a test PC can be identified and selected for sequential tests; this manual does not claim concurrent acquisition qualification.

Keep the complete extracted SDK, including dependencies and license files. Firmware images are supplied separately. Specialized application notes and instrument specifications complement this SDK manual.
