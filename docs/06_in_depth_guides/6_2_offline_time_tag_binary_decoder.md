# 6.2 Offline time-tag binary decoder

The native reader decodes NXTT files without requiring a device handle. Python wraps its batch reader and writes optional CSV with Python's standard CSV module.

```python
# library was loaded from the matching extracted SDK; acquisition is finished.
with library.open_time_tag_file_reader("capture.nxtt") as reader:
    header = reader.header()  # Host representation, not bytes to unpack with sizeof.
    written = reader.export_csv("capture.csv")
    print(written)  # Zero is empty input, not evidence of a successful capture.
```

For custom processing use `read_batch(max_tags=4096)` or `iter_tags(batch_size=4096)`. Timestamps are integer picoseconds; preserve integer arithmetic before converting units. An exception is a read failure. In C, only success with zero returned tags means EOF.

## Rotated series

Use the series reader with directory, `series_key`, first sequence and last sequence. The key is `base[_timestamp]_TAGS[_suffix]`, without the injected four-digit sequence. Sequences are one-based; `seq_end=0` means all remaining contiguous files. Missing required segments must be reported rather than silently skipped. The primary raw template enumerates/decodes its complete capture and retains file provenance.

## When a device is connected

Native rejects offline decoding while tag output or file saving remains active. Otherwise it can temporarily establish quiet output and restore the prior mode when the reader releases its ownership. EOF releases gating automatically; still close the reader. A mode-restore failure is an error, not normal EOF.

Do not overlap raw acquisition and decoding on the same native module. Finalize capture first, and do not use a saved file's timestamps as proof of a new physical clock origin or globally synchronized devices.

[In-depth guides](index.md) · [C reader API](../07_c_api/7_18_binary_decoder.md)
