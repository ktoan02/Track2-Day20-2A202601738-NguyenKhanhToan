# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=6` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 199 | 3.36 | 1700 | 3900 | 7200 | 6.8 | 5.0% |
| 50 | 229 | 3.87 | 12000 | 13000 | 13000 | 41.8 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.15x** (23% of linear) |
| P95 latency | **3.33x** |
| Effective concurrency at 50 users | 41.8 vs `--parallel 4` slots (occupancy/slot ratio 10.46) |

**Saturated.** Throughput delivered only 1.15x for 5x the offered load, and effective concurrency (41.8) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.15x while P95 moved 3.33x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

### Saturation Analysis & SLO Optimization

1. **Saturation Threshold & Convincing Evidence**:
   - The server saturates around **10 to 12 concurrent users**.
   - Increasing load by 5x (from 10 to 50 users) only yields a **1.15x throughput increase** (3.36 to 3.87 RPS), while **P95 latency explodes 3.33x** (3,900 ms to 13,000 ms).
   - At 50 users, the effective concurrency (41.8) is **10.46x the available slots** (--parallel 4), showing that ~90% of user latency is queue time rather than compute time.

2. **Knob Recommendation for SLO Optimization**:
   - If the SLO target is P95 < 4.0s, maximum goodput occurs around 10 users (RPS ~3.36).
   - To raise goodput at this SLO, the primary knob to change is **increasing --parallel from 4 to 8 slots**.
   - Since the model (0.8B) is tiny and fits easily in VRAM, increasing parallel slots allows the batching scheduler to process more concurrent streams simultaneously without queuing delay.