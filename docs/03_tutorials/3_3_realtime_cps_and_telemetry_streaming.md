# 3.3 Realtime CPS and telemetry streaming

```sh
# Observe callbacks through the complete example's setup and cleanup.
python python/examples/ftdi_realtime_smoke.py --home . --duration-sec 10
```

Use an input source for nonzero CPS; a connected device with no events can legitimately report zero. The example handles runtime readiness and the needed measurement controls.

For custom telemetry polling, register a callback and use the native request API on an authorized telemetry-capable profile:

```python
# Fragment inside a connected device context; return quickly from callbacks.
latest = {}
def on_telemetry(view):
    latest["telemetry"] = view  # Python's public callback wrapper supplies an owned copy.
device.set_telemetry_view_callback(on_telemetry)
device.request_telemetry()  # Request accepted; a later callback is the response.
```

Do not assume `enable_telemetry(True)` produces periodic reports in quiet mode. A view getter reads cached data; inspect freshness and field availability. CPS `counts` already contains Hz, including for a 100 ms period.

Keep callback references until native device destruction completes; clearing the registration alone does not retire an already-selected invocation. Perform expensive plotting or file processing outside callbacks and surface callback exceptions in your application. See [callback lifetime](../06_in_depth_guides/6_5_callback_thread_safety_and_data_lifetime.md).

[Tutorials](index.md)
