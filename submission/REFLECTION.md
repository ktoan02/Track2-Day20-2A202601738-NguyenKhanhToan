# Reflection — Day 20 Lab (Personal Report)

**Họ Tên:** Nguyễn Khánh Toàn
**Cohort:** 4
**Ngày submit:** 2026-08-21

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

- **OS:** Windows 11 (AMD64)
- **CPU:** 11th Gen Intel(R) Core(TM) i5-11400H @ 2.70GHz
- **Cores:** 6 physical / 12 logical
- **CPU extensions:** AVX2 / FMA / SSE4.2
- **RAM:** 15.8 GB
- **Accelerator:** NVIDIA GeForce RTX 3050 Laptop GPU (4096 MiB VRAM)
- **llama.cpp asset đã tải:** llama-b10488-bin-win-cuda-12.4-x64.zip
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M (primary) + UD-Q2_K_XL (compare)

**Chạy ở đâu:** Laptop cá nhân của tôi.

**Setup story**: Máy có GPU RTX 3050 4GB VRAM và 16GB RAM. Em lựa chọn model Qwen 3.5 0.8B để tối ưu hóa bộ nhớ và tốc độ xử lý. Khi chạy trên Windows PowerShell, lệnh verify gặp lỗi bảng mã UTF-8 với ký tự đặc biệt, em đã khắc phục bằng cách đặt PYTHONIOENCODING=utf-8 và chuẩn hóa đường dẫn file.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
| ------------ | --------: | --------: | ----------------: | ----------------: | -------------------: | -------------: |
| Q4_K_M       |      0.50 |      2790 |         204 / 266 |         6.3 / 7.2 |      600 / 662 / 662 |          157.5 |
| UD-Q2_K_XL   |      0.39 |      1835 |         199 / 207 |         6.7 / 6.9 |      622 / 636 / 636 |          150.2 |

**Quan sát**: Bản 2-bit (UD-Q2_K_XL) chậm hơn bản 4-bit (Q4_K_M) khoảng 1.05 lần (150.2 vs 157.5 tok/s). Lý do là model 0.8B quá nhỏ và nằm hoàn toàn trong VRAM GPU, nên không bị nghẽn băng thông bộ nhớ (memory bandwidth). Việc giải nén toán học 2-bit phức tạp làm tăng overhead tính toán của GPU kernel. Do đó bản 4-bit Q4_K_M vượt trội cả về tốc độ lẫn chất lượng câu trả lời.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

| Users |  RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
| ----: | ---: | -------: | -------: | -------: | ---------------: | -------: |
|    10 | 3.36 |     1700 |     3900 |     7200 |              6.8 |     5.0% |
|    50 | 3.87 |    12000 |    13000 |    13000 |             41.8 |     0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.15×
- **P95 tăng:** 3.33×
- **Effective concurrency ở 50 users:** 41.8 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`**: 4.00 / 4 slots

**Saturation reading**: Server bão hòa ở mức khoảng 10-12 concurrent users. Bằng chứng là khi tăng tải từ 10 lên 50 users (tăng 5x), throughput chỉ tăng 1.15x (từ 3.36 lên 3.87 RPS) trong khi P95 latency bùng nổ gấp 3.33x (từ 3.9s lên 13.0s). Do số slot tối đa là `--parallel 4`, 41.8 effective concurrency nghĩa là trung bình ~37.8 request bị nghẽn trong hàng đợi (queue time). Để tăng goodput@SLO (ví dụ SLO P95 < 4s), em sẽ tăng `--parallel` từ 4 lên 8 slots trước tiên vì VRAM GPU RTX 3050 còn thừa rất nhiều cho model 0.8B.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

| Day                   | Piece                  | Real hay stub? |
| --------------------- | ---------------------- | -------------- |
| N16 Cloud/IaC         | Native Local Execution | real           |
| N17 Data pipeline     | Python Script Data     | stub           |
| N18 Lakehouse         | Memory Text Chunks     | stub           |
| N19 Vector + features | Keyword BM25 Matching  | stub           |
| N20 Serving           | `llama-server`       | real           |

**Latency split**:

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 4065.8 ms
- **stage chiếm nhiều nhất:** llm (100.0% của total)

**Reflection**: Bottleneck nằm hoàn toàn ở stage LLM generation (chiếm 100.0% tổng thời gian). Đúng như kỳ vọng vì việc suy luận LLM phức tạp hơn nhiều so với tìm kiếm từ khóa (0.1ms). Để giảm latency pipeline 2x, em sẽ tấn công vào stage LLM bằng cách bật Prompt Caching (KV cache reuse) để tái sử dụng trạng thái KV của phần context/prompt chung giữa các truy vấn.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

**Change:** Lựa chọn bản Quantization phù hợp (`UD-Q2_K_XL` sang `Q4_K_M`) khi offload GPU.

```
before:  150.2 tok/s (UD-Q2_K_XL)
after:   157.5 tok/s (Q4_K_M)
speedup: 1.05×
```

**Tại sao nó work**:
Thay đổi này mang lại hiệu quả vì bản chất bottleneck phần cứng của máy em khi chạy model 0.8B offload GPU. Trên mô hình nhỏ 0.8B (0.50 GB), toàn bộ trọng số được nạp trọn vẹn vào VRAM tốc độ cao của RTX 3050 GPU, loại bỏ hoàn toàn tình trạng nghẽn băng thông bộ nhớ (memory bandwidth bound).

Tại điểm này, quá trình decode trở thành Compute-bound / GPU Kernel Overhead-bound. Bản nén 2-bit Unsloth Dynamic (`UD-Q2_K_XL`) tuy tiết kiệm được ~110MB VRAM nhưng yêu cầu các phép toán giải nén bit-packing phi tuyến tính phức tạp hơn nhiều so với chuẩn 4-bit (`Q4_K_M`). Chi phí tính toán dequantization tăng lên của 2-bit lớn hơn lợi ích tiết kiệm dung lượng, dẫn đến việc bản 4-bit Q4_K_M cho tốc độ sinh chuỗi nhanh hơn 1.05x đồng thời giữ được chất lượng câu trả lời cao hơn hẳn.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

**Đã làm:** N/A

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Điều ngạc nhiên nhất là bản nén 2-bit không phải lúc nào cũng nhanh hơn bản 4-bit. Khi offload GPU và bộ nhớ đủ đáp ứng, chi phí tính toán giải nén (dequantization overhead) có thể làm chậm tốc độ suy luận hơn so với bản nén vừa phải 4-bit.

---

## 8. Self-check trước khi push

- [X] `hardware.json` committed
- [X] `models/active.json` committed
- [X] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [X] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [X] `benchmarks/02-server-results.md` committed (`make load-report`)
- [X] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [X] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [X] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [X] Mọi section "required — replace this line" trong các file `benchmarks/*.md` đã được thay bằng nhận xét của bạn
- [X] 5 screenshots trong `submission/screenshots/`
- [X] `make verify` → **exit 0**
- [X] Repo GitHub ở chế độ **public**
- [X] Đã paste public URL vào VinUni LMS
- [X] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)
