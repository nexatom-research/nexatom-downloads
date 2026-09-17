# 2.11 Telemetry

## Telemetry modes

Telemetry capability and field layout depend on the resolved runtime. `enable_telemetry(True)` configures periodic reporting where supported; it does not guarantee callbacks in every output mode. Current common-runtime periodic reporting requires processed output. Explicit supported requests can respond in quiet output mode without starting acquisition.

## Requesting telemetry on-demand

Use `request_telemetry()` / `nexatom_tt_request_telemetry`. Native encodes the appropriate request for the authorized runtime. Successful return accepts the request; observe a subsequent callback or view revision for a new sample instead of sleeping for an assumed response time.

## Polling latest telemetry

`get_telemetry_view()` returns the latest available versioned view, or `None` when no view is available. It is a cache read, not a fresh device transaction. `get_telemetry()` is the legacy request/wait API and returns the fixed legacy record. Prefer a request plus callback/view when managing freshness explicitly.

Use availability indicators before interpreting version-specific fields. Keep received time, source version and sequence; sequences can wrap. The legacy `system_status_word` is not the entire common-runtime status payload and cannot establish a stopped output mode. The numeric telemetry serial is not the complete FT601 selection string.

See [C telemetry](../07_c_api/7_15_telemetry.md) and [Python callbacks](../05_api_reference/5_4_callback_types.md).

[Device operation](index.md)
