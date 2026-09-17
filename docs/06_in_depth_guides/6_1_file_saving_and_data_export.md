# 6.1 File saving and data export

## Choosing the output product

Raw/encoded time-tag saving supports the native time-tag formats, including binary NXTT. Processed saving supports CSV, TAB and, in the published package, HDF5. Choose supported output modes and packet types from the active profile.

Enabling time-tag saving disables processed saves; enabling processed saving disables tag saves. Processed CSV/TAB and processed HDF5 cannot be mixed in one active configuration. File saving does not start an acquisition. Configure while quiet, explicitly start the measurement, observe required terminal results, then quiet output and finalize the sinks.

## NXTT binary format

NXTT is a little-endian **decoded time-tag file**, not the instrument's USB wire stream:

| Disk element | Serialized bytes |
| --- | --- |
| Header magic | 4 bytes, ASCII `NXTT` |
| Creation timestamp | uint64, microseconds since epoch |
| Output data type | uint32, encoded/raw mode |
| Each block count | uint32 number of following tags |
| Each tag | uint64 timestamp in ps, then uint8 channel, without padding |

The disk header is **16 bytes** and each tag within a block is **9 bytes**. The C header and tag structures are aligned host representations; do not write/read them with `sizeof` as a disk codec. Prefer the supplied native reader.

## Processed formats

Processed text timestamps are Unix epoch milliseconds. CPS columns are rates in Hz. TIHI includes histogram settings/statistics and bins; MFCO includes pattern settings/counts; correlation stores lag/value pairs. Preserve packet-specific metadata and schema/producer identity.

CSV is not generally safe to parse by splitting a string on commas. Where marked MFCO CSV v1 is present, skip leading `#` preamble lines, decode CSV quoting, then parse `metadata_json`; expect six fixed fields plus the declared pattern count. Historical unversioned MFCO and legacy TIHI/CORL/CORM CSV metadata have different framing. Use a format-specific parser or the documented TAB/HDF5 representation; do not silently treat an old file as a new schema.

For HDF5, inspect root `schema_version` and dataset columns/units/description attributes. Version 3 MFCO carries additional result metadata; absent metadata in older files remains unavailable. Do not invent zero-valued quality evidence. Specialized Fast TIHI/configuration products likewise require their own packet/schema identity; a `.csv` extension does not establish a common row layout. Consult the package's C API guide for exact dataset/record schemas.

## Naming, rotation and finalization

`base_filename` can include an output directory. Native creates directories and commonly date subdirectories; sequence numbers are inserted into stream filenames. Configurations provide size, duration, event-count and supported acquisition-boundary rotation, timestamp inclusion and a suffix. HDF5 uses a session container rather than per-packet rolling limits. Do not assume a file exists immediately after enabling an empty sink.

`flush_immediately` affects file-manager flushing, not a durable-storage guarantee or proof that acquisition has finished. Check final files and cleanup errors. Keep the [raw](../03_tutorials/3_4_raw_time_tag_capture_and_offline_csv_export.md) or [processed](../03_tutorials/index.md#processed-and-raw-template-anatomy) template's stop/finalize order.

[In-depth guides](index.md)
