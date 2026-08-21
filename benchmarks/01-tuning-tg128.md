# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **6 physical · 12 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 165.8 | 99% |
| 3 | 166.3 | 99% |
| 6 | 164.2 | 98% |
| 12 | 165.5 | 99% |
| 24 | 167.4 | 100% |

**Best**: `-t 24` at 167.4 tok/s
**Slowest tested**: `-t 6` at 164.2 tok/s (1.02x spread)
**Against the physical-core default** (`-t 6`, 164.2 tok/s): 1.02x

Use this in your run:

```bash
LAB_N_THREADS=24 make bench
```

## Your explanation

### Analysis & Observations

1. **Curve Shape & Spread**:
   - Across all tested thread counts (-t 1, 3, 6, 12, 24), decode speed remains virtually identical, ranging from **164.2 tok/s** (-t 6) to **167.4 tok/s** (-t 24). The maximum speed difference across the entire sweep is only **~2%**.

2. **Root Cause Analysis**:
   - Since GPU layer offloading is active (
gl=99), all neural network layer matrix multiplications and KV cache lookups are executed on the dedicated NVIDIA RTX 3050 GPU VRAM.
   - CPU threads are only responsible for lightweight dispatching and API scheduling. As a result, CPU core saturation or thread over-subscription does not bottleneck generation throughput, causing the performance curve across CPU thread counts to remain completely flat.

3. **Recommendation**:
   - Setting -t 6 (matching physical CPU cores) or keeping default thread count is ideal to avoid wasting host CPU cycles on idle worker polling while leaving CPU capacity free for HTTP request processing and system tasks.