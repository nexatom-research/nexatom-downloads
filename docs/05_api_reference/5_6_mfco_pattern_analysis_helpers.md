## MFCO Pattern Analysis Helpers

When processing Multi-Fold Coincidence (MFCO) data, the hardware returns a 256-element array (`pattern_bins`) where each index represents an 8-bit channel bitmask. For example, index `5` (binary `00000101`) corresponds to a simultaneous coincidence on Channel 0 and Channel 2.

To avoid repeatedly implementing bitwise masking logic in Python, the SDK provides the small `nexatomtt.analysis` module. It operates on the eight-bit, 256-bin MFCO representation; this does not imply that every product has only eight physical TDC resources.

**Usage:**
```python
from nexatomtt.analysis import top_patterns, exact_pattern_count
```

#### Analysis Functions

| Function | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `pattern_channels` | `pattern_index: int` | `list[int]` | Decodes an 8-bit pattern index (0-255) into a list of active hardware channels (e.g., `5` $\rightarrow$ `[0, 2]`). |
| `pattern_mask` | `channels: list[int]` | `int` | Encodes a list of channels into the corresponding 8-bit pattern index bitmask. |
| `exact_pattern_count` | `pattern_bins: list[int]`, `channels: list[int]` | `int` | Returns the count of coincidences where *only* the specified channels fired (strict exact match). |
| `contains_channels_count`| `pattern_bins: list[int]`, `required_channels: list[int]` | `int` | Returns the aggregated sum of all coincidence patterns that include the specified channels, regardless of what other channels also fired (superset match). |
| `order_counts` | `pattern_bins: list[int]` | `dict[str, int]` | Aggregates the entire histogram and groups the counts by order: `singles`, `doubles`, `triples`, and `higher`. |
| `top_patterns` | `pattern_bins: list[int]`, `limit: int = 10` | `list[dict]` | Returns nonzero-count records, sorted by descending count then ascending pattern for ties. |

### Exact and inclusive coincidence counts

```python
from nexatomtt.analysis import exact_pattern_count, contains_channels_count

# result is one MFCO callback record whose status/quality you have checked.
bins = [int(value) for value in result.pattern_bins]
only_0_and_1 = exact_pattern_count(bins, [0, 1])
includes_0_and_1 = contains_channels_count(bins, [0, 1])
# The inclusive result also counts patterns with channel 2, 3, etc. present.
```

Both helpers require exactly 256 nonnegative integer counts. Channel IDs must be integers in 0–7. `order_counts` excludes the empty pattern and groups the remaining bins by the number of set bits. These helpers neither validate completion/quality nor divide by acquisition live time. Inspect those result fields before interpreting a rate, and do not add successive `WHOLE_RUN` results together.
