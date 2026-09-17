# 2.8 External clock input

## Requesting external sync-clock operation

Check the active profile/capabilities before `request_sync_clock(enable)` or `nexatom_tt_request_sync_clock`. A successful request is distinct from detecting a reference and locking to it.

Use the reference frequency, voltage, termination and cabling specified for your instrument. This SDK manual does not establish those physical limits or promise synchronisation between two boards merely because both are connected to the same PC.

## Sync-clock status monitoring via telemetry

Use the versioned telemetry view and its field availability indicators. Keep requested, detected/locked and active clock state distinct. A missing field is unknown, not false; a cached sample may be stale. Explicitly request a newer sample when necessary.

Clock lock alone does not measure relative board skew or prove that two acquisitions share a timestamp origin. Such experiments need their own measured acceptance criteria.

[Device operation](index.md) · [Telemetry](2_11_telemetry.md)
