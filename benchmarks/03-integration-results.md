# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 19414.6 | 19414.9 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.3 | 6586.5 | 6587.0 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.2 | 13834.9 | 13835.3 |

Mean per stage (ms): embed **0.0** · retrieve **0.2** ·
llm **13278.7** · total **13279.1**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the provided context, **goodput is more useful than raw throughput** because goodput focuses on the actual requests per second that met the Target Time-to-Fill (TTFT) and Target Time-to-Poll (TPOT) targets, whereas raw throughput ignores SLOs (Service Level Objectives).

The context explicitly states: "Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Thr

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation** in GPU memory by storing the KV cache in non-contiguous pages, which allows for more efficient memory usage compared to contiguous arrays.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when the **prefill operation is compute-bound** and the **decode operation is memory-bandwidth-bound**.

This is because the context explicitly states that prefill is compute-bound (requiring significant CPU work) and decode is memory-bandwidth-bound. By splitting these tasks, the system avoids the need for a single large prefill pass, allowing the engine to skip


## Which N16-N19 pieces are real

- **N16 Cloud/IaC:** stub (chạy trực tiếp trên local Windows laptop)
- **N17 Data pipeline:** stub (dùng tài liệu mẫu `TOY_DOCS`)
- **N18 Lakehouse:** stub (in-memory list)
- **N19 Vector + features:** stub (sử dụng thuật toán keyword overlap fallback do không bật server embedding riêng)
- **N20 Serving:** **real** (kết nối trực tiếp tới `llama-server` qua endpoint OpenAI-compatible `/v1/chat/completions`)

**Nhận xét:**
Stage chiếm ưu thế tuyệt đối là **LLM (100% tổng độ trễ, trung bình 13278.7 ms / 13279.1 ms)**, hoàn toàn khớp với kỳ vọng vì retrieval dạng keyword chỉ mất 0.2 ms trong khi LLM phải thực hiện prefill context dài và decode tuần tự từng token.
Để giảm độ trễ pipeline 2x, chắc chắn phải tấn công vào **stage LLM** bằng cách:
1. **Bật Prompt Caching (Prefix Caching):** Tái sử dụng KV cache của phần system prompt và văn bản truy xuất nhằm triệt tiêu gần như hoàn toàn thời gian prefill.
2. **Speculative Decoding:** Dùng draft model hoặc MTP head để giải mã nhiều token song song mỗi bước forward, tăng tốc độ decode.
