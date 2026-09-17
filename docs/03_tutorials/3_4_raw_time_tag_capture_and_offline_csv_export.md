# 3.4 Raw time-tag capture and offline CSV export

```sh
# Use an internal source, capture a new run, and export every captured file.
python python/examples/file_save_and_offline_decode.py --home . --output-dir captures --duration-sec 5 --internal-test --decode-csv
```

The template validates input settings and RAW_TAGS support, prepares native file saving, starts acquisition and records a separate run directory. It then stops/quietens the source, finalizes native sinks and decodes the complete NXTT file set. Missing files, empty decoded data, decoding failures and cleanup failures are not successful captures.

The summary records channel counts and intervals. An expected internal-pulse interval is useful evidence, but finite host capture boundaries and scheduling are not an exact physical acquisition gate. Keep the original NXTT files and summary when exporting CSV.

NXTT is a serialized native time-tag file format; it is not a byte-for-byte USB dump or an array of aligned C structures. See [file formats](../06_in_depth_guides/6_1_file_saving_and_data_export.md).

C/C++ raw templates in `examples/sdk/` follow the same stages. Use their README for target names and compiler commands, and preserve their error-aware cleanup when adapting the source.

[Tutorials](index.md) · [Offline reader](../06_in_depth_guides/6_2_offline_time_tag_binary_decoder.md)
