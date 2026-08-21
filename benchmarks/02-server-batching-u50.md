# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 15 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 4.00 of 4 slots (100%) |
| `requests_processing` | 4 |
| `requests_deferred` | 44 |
| `kv_cache_usage_ratio` | n/a  not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 15030 |

Highest sampled value was **4.00 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

### Analysis & Observations

1. **Peak Batch Width & Continuous Batching Evidence**:
   - The peak sampled n_busy_slots_per_decode reached 4.00 of 4 slots (100% capacity) during the 50-user load test.
   - requests_processing remained at 4 while requests_deferred peaked at 44 queued requests, proving continuous batching was fully packed.

2. **Agreement between Metrics & Little Law**:
   - make metrics measures true slot utilization inside the engine (4 active slots).
   - 02-server-results.md measures total effective concurrency via Little Law (41.8 in-flight requests).
   - Both metrics align: 4 requests are actively processed in slots while ~37.8 requests wait in queue.