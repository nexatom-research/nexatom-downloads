# 5.2 NexatomTT classes

## NexatomLibrary

| Method | Behaviour |
| --- | --- |
| `discover_devices(max_devices=NEXATOM_MAX_DEVICES_DEFAULT)` | Return enumerated device records |
| `create_device(device_info)` | Create an owning `NexatomDevice`; use a context manager |
| `version()`, `library_info()` | Native version/build identity |
| `open_time_tag_file_reader(path)` | Open one NXTT file for offline reading |
| `open_time_tag_series_reader(...)` | Open a named contiguous file series; see packaged signature |

## NexatomDevice

| Method group | Main operations |
| --- | --- |
| Startup | `connect_runtime(timeout_ms=5000)`, `disconnect()`, `close()` |
| Inspection | `is_connected()`, `state()`, `hardware_protocol_mode()`, `get_device_info()` |
| Profile | `get_device_profile()`, `get_capabilities()`, `get_application_contract()` |
| System | `enable_system(enable)`, `set_output_type(output_type)`, `get_output_type()`, `reset_peripherals(reset)`, `request_global_stop_all_modes()` |
| Telemetry | `request_telemetry()`, `get_telemetry()`, `get_telemetry_view()` |
| File saving | `set_time_tag_file_config(config)`, `enable_time_tag_file_saving(base_filename, format=...)`, `disable_time_tag_file_saving()` |
| Processed saving | `set_processed_file_config(packet_type, config)`, `enable_processed_file_saving(packet_type, base_filename, format=...)`, `disable_processed_file_saving(packet_type)` |
| Service | `request_field_upgrade_service_entry()`, `refresh_field_update_status()`, `load_field_update_image(...)`, `boot_field_update_slot(slot_index)` |
| Persistent default | `set_field_update_default_slot(slot_index)`; only when intentionally requested |

`connect()` opens/detects a device without replacing the measurement-readiness contract of `connect_runtime()`. Ordinary applications should use native runtime startup. Manual/inventory profile evidence APIs are advanced controlled compatibility inputs, not required startup configuration or permission to claim capabilities.

## RuntimeBootOptions and open_runtime_device

```python
# A single handle survives any required service-to-runtime boot.
options = RuntimeBootOptions(timeout_ms=20000)
with open_runtime_device(library, device_info, options=options) as device:
    profile = device.get_device_profile()  # Read authority before choosing a measurement.
```

The optional `preferred_slot` applies when a service-mode boot is needed. An existing runtime still passes native readiness on the same handle. Legacy polling options remain helper compatibility settings; do not use them to implement your own firmware protocol.

## NexatomTimeTagReader

Use a context manager or `close()`. `header()` returns a host record; `read_batch(max_tags=4096)` returns a batch and an empty batch means EOF. `iter_tags(batch_size=4096)` yields decoded tags. `export_csv(output_path, batch_size=4096)` returns the number written using Python's CSV writer over native-decoded batches. Exceptions remain errors, not EOF.

[Python reference](index.md) · [Offline reader](../06_in_depth_guides/6_2_offline_time_tag_binary_decoder.md)
