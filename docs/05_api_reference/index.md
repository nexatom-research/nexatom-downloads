# 5. Python API reference

The packaged `python/nexatomtt` module wraps the public C API and owns its ctypes layouts/callback copies. Use that package with its matching native library. This chapter describes preview.7; see [migration scope](../index.md#version-scope-and-migration).

- [5.1 Library and constants](5_1_nexatomtt_library.md)
- [5.2 Classes and lifecycle](5_2_nexatomtt_classes.md)
- [5.3 Data structures](5_3_data_structures.md)
- [5.4 Callbacks](5_4_callback_types.md)
- [5.5 Measurement and analysis controls](5_5_measurement_modules.md)
- [5.6 MFCO pattern helpers](5_6_mfco_pattern_analysis_helpers.md)

Begin with the [annotated templates](../03_tutorials/index.md#processed-and-raw-template-anatomy). Native remains the authority for capability and parameter validation. Python type/range checks prevent integer wrapping; they do not turn unsupported hardware into a supported profile.

[Manual contents](../index.md)
