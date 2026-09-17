# 6.3 Logging system

Logging is process-global for the loaded native module. `set_log_callback` installs a structured callback; it is separate from device measurement callbacks. Use the supplied enum constants and query logging capabilities rather than assuming every build includes every level/category.

`NexatomLogRecord` / `nexatom_log_record_t` contains an ISO-8601 UTC `timestamp`, module, level, category, sequence ID, repetition count, message length and copied UTF-8 message. Keep callbacks short: delivery uses a bounded queue and slow consumers can lose diagnostic records. A log callback is not a lossless acquisition channel.

The library exposes capabilities, requested/effective configuration and statistics. Statistics include submitted/emitted/suppressed/dropped records and queue occupancy. Configuration can filter modules, levels and categories. Do not infer a file-rotation facility, particular solver trace or zero overhead merely from these controls.

```python
# library is a NexatomLibrary; collect diagnostic text outside the hot data path.
records = []
library.set_log_callback(lambda record: records.append(record))
try:
    print(library.version())  # Replace with the bounded operation being diagnosed.
finally:
    library.unregister_log_callback(timeout_ms=5000)  # Wait before releasing handler state.
```

Unregister from the owning control thread, not from a native callback. On failure/timeout retain resources and handle the error. `clear_callbacks()` on a device does not unregister the process-global logger.

[In-depth guides](index.md) · [C logging](../07_c_api/7_2_logging_config.md)
