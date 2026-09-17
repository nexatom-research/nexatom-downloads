# 3.1 Library load and version check

```sh
# Load the matching Windows DLL or Linux shared library without opening hardware.
python python/examples/version_info.py --home .
```

The example locates the extracted package, loads its dependencies and prints native version/build identity. Compare identity with the package you intended to use. The native version string and SDK release version are separate values; do not test for a preview label in `version()`.

If loading fails, verify host architecture, the complete package and dependency search paths. Do not replace just one FTDI or compiler runtime. Successful loading does not prove USB access or runtime readiness; continue to [runtime startup](3_2_booting_into_runtime_from_bootloader.md) or the [processed template](index.md#processed-and-raw-template-anatomy).

[Tutorials](index.md)
