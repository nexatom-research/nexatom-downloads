## MFCO Pattern Analysis Helpers

When processing Multi-Fold Coincidence (MFCO) data, the hardware returns a 256-element array (`pattern_bins`) where each index represents an 8-bit channel bitmask. For example, index `5` (binary `00000101`) corresponds to a simultaneous coincidence on Channel 0 and Channel 2.

To prevent developers from having to manually implement bitwise masking logic in Python, the SDK provides the `nexatomtt.analysis` module containing highly optimized helper functions.

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
| `top_patterns` | `pattern_bins: list[int]`, `limit: int` | `list[dict]` | Returns the top N most frequent non-zero coincidence patterns, sorted in descending order by count. |
