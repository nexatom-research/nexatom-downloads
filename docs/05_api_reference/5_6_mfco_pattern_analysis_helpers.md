# 5.6 MFCO pattern analysis helpers

`nexatomtt.analysis` provides small Python helpers for the legacy 256-bin, eight-bit MFCO pattern representation. Pattern 5 has bits 0 and 2 set. This representation is not a statement that every physical device has exactly eight TDC lanes.

| Function | Meaning |
| --- | --- |
| `pattern_channels(pattern)` | Return set channel bits for an integer pattern 0–255 |
| `pattern_mask(channels)` | Combine channel IDs 0–7 into a bitmask |
| `exact_pattern_count(pattern_bins, channels)` | Count the one pattern with exactly those bits |
| `contains_channels_count(pattern_bins, required_channels)` | Sum all patterns containing those bits, including additional channels |
| `order_counts(pattern_bins)` | Group singles, doubles, triples and higher orders; excludes the empty pattern |
| `top_patterns(pattern_bins, limit=10)` | Return nonzero-count pattern/count records, descending count then ascending pattern |

```python
from nexatomtt.analysis import exact_pattern_count, contains_channels_count

# Read one valid result; do not sum overlapping ACCUMULATE snapshots.
bins = [int(value) for value in result.pattern_bins]
only_0_and_1 = exact_pattern_count(bins, [0, 1])
includes_0_and_1 = contains_channels_count(bins, [0, 1])
# The inclusive value also counts events containing additional channels.
```

Helpers validate the pattern range and exactly 256 non-negative integer counts. They do not validate acquisition completion, raw hardware error flags, host quality or normalization by live time; inspect those on the result first. A host wall-clock duration is not automatically the correct MFCO rate denominator.

[Python reference](index.md) · [MFCO C reference](../07_c_api/7_10_mfco.md)
