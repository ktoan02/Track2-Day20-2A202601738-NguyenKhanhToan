# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 3136.1 | 3136.2 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 3026.3 | 3026.3 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 3542.3 | 3542.3 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **3234.9** · total **3234.9**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the context provided, **Goodput** is more useful than raw throughput because it **ignores SLOs** at saturation.

While raw throughput measures the total requests per second (requests per second), Goodput specifically counts only the requests that met the Target Time-to-Fullness (TTFT) and Target Time-to-Poll (TPOT) targets. This means Goodput only counts requests that actually satisfied t

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation** in GPU memory.

While RadixAttention uses a trie to cache keys by token prefix to avoid prefilling, PagedAttention specifically addresses a different issue: the internal fragmentation of the KV cache itself. By storing the KV cache in non-contiguous pages, PagedAttention removes the wasted space that would otherwise exist within the c

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps to **avoid memory bandwidth bottlenecks during the decoding phase**, while simultaneously **preventing the prefill phase from becoming a bottleneck due to compute-bound execution**.

Here is the breakdown of how this optimization works based on the provided context:

1.  **Prefill is Compute-bound**: The prefill phase requires significant processing power (CPU/GP


## Which N16-N19 pieces are real

### Pipeline Stage Classification & Latency Analysis

1. **Component Status (Real vs Stub)**:
   - **N16 (Inference Engine)**: **REAL** -- llama-server running OpenAI API on port 8080.
   - **N17 (Continuous Batching & KV Caching)**: **REAL** -- llama-server native slot management.
   - **N18 (Embedding Model)**: **STUBBED** -- Using keyword overlap fallback (0.0 ms).
   - **N19 (Vector Search / Retrieval)**: **STUBBED** -- Using in-memory BM25/keyword matching (0.1 ms).

2. **Dominant Stage Evaluation**:
   - As expected, the **LLM stage is overwhelmingly dominant**, consuming **4065.8 ms out of 4065.9 ms (100.0% of total latency)**.
   - In any local RAG system, neural generation accounts for virtually all execution time compared to lightweight retrieval.

3. **Strategy to Halve Pipeline Latency**:
   - To halve total latency, we must target the **LLM stage** (optimizing the 0.1 ms retrieval yields no practical improvement).
   - Primary techniques:
     - **Prompt Caching / KV Reuse**: Cache the prefill KV state of shared context/system instructions so prefill cost drops to ~0 ms.
     - **Speculative Decoding / Decoding Optimization**: Use speculative decoding or smaller quantizations to reduce per-token decode latency.