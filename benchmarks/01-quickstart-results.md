# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=6` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 2790 | 204 / 266 | 6.3 / 7.2 | 600 / 662 / 662 | 157.5 |
| UD-Q2_K_XL | 0.39 | 1835 | 199 / 207 | 6.7 / 6.9 | 622 / 636 / 636 | 150.2 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.05x SLOWER** than `Q4_K_M` here, despite being 0.11 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead  few cores, no GPU offload  the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

### Analysis & Observations

1. **Speed vs Size Trade-off**:
   - Q4_K_M (0.50 GB) achieves a decode speed of **157.5 tok/s** with a P50 TPOT of **6.3 ms**.
   - UD-Q2_K_XL (0.39 GB) achieves **150.2 tok/s** with a P50 TPOT of **6.7 ms**.
   - UD-Q2_K_XL is **~4.6% (1.05x) slower** in decode speed despite saving 0.11 GB of VRAM.

2. **Root Cause (Compute-bound vs Bandwidth-bound)**:
   - Since all layers are offloaded to GPU (
gl: 99), memory bandwidth is plentiful for a tiny 0.8B model (0.50 GB).
   - The decode phase is compute/kernel-overhead bound rather than VRAM bandwidth bound. The complex non-uniform dequantization arithmetic required for 2-bit Unsloth Dynamic (UD-Q2_K_XL) adds compute overhead that outweighs the 110 MB bandwidth saving.

3. **Conclusion & Recommendation**:
   - On this machine, Q4_K_M is superior in both speed (157.5 tok/s vs 150.2 tok/s) and generation quality. UD-Q2_K_XL is not worth using unless VRAM is extremely constrained.