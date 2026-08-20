# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Trần Đức Thiện
**Cohort:** AICB-P2T2 (2A202602032)
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11
- **CPU:** 11th Gen Intel(R) Core(TM) i5-1135G7 @ 2.40GHz
- **Cores:** 4 physical / 8 logical
- **CPU extensions:** AVX2, AVX-512
- **RAM:** 7.7 GB
- **Accelerator:** Vulkan (Intel Iris Xe Graphics)
- **llama.cpp asset đã tải:** llama-b10488-bin-win-vulkan-x64.zip
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Môi trường Windows PowerShell 5.1 cần đồng bộ bảng mã UTF-8 cho script runner console và cấu hình lại encoding cho stdout/file IO để hiển thị và trích xuất dữ liệu metrics chính xác.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 15879 | 710 / 866 | 50.7 / 59.8 | 3902 / 4518 / 4518 | 19.7 |
| UD-Q2_K_XL | 0.39 | 11980 | 1159 / 1410 | 271.0 / 300.6 | 18273 / 20134 / 20134 | 3.7 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

Bản 2-bit giải mã chậm hơn 5.32x (3.7 vs 19.7 tok/s) và câu trả lời bị suy giảm chất lượng rõ rệt. Mức tiết kiệm 0.11 GB RAM không bù đắp được chi phí dequantization lớn trên CPU/iGPU, hoàn toàn không đáng dùng.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.23 | 34000 | 43000 | 43000 | 7.2 | 0.0% |
| 50 | 0.31 | 41000 | 52000 | 52000 | 11.4 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.35×
- **P95 tăng:** 1.21×
- **Effective concurrency ở 50 users:** 11.4 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.80 / 4 slots

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Server bão hòa ở giữa 10-50 users: RPS tăng dưới tuyến tính (1.35x khi tải tăng 5x) và `requests_deferred` đạt 46 với concurrency 11.4 > 4 slots, chứng minh độ trễ tăng chủ yếu là queue time. Để nâng goodput@SLO, tôi sẽ tăng `--parallel` lên 8 slot trước tiên.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Localhost Windows | stub |
| N17 Data pipeline | In-memory TOY_DOCS | stub |
| N18 Lakehouse | In-memory list | stub |
| N19 Vector + features | Keyword overlap | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.2 ms
- llm: 13278.7 ms
- **stage chiếm nhiều nhất:** llm (100.0% của total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

Bottleneck nằm 100% ở stage LLM (13.3s vs 0.2ms retrieval). Để giảm latency 2x, cần tấn công vào LLM bằng Prefix Caching (RadixAttention) nhằm loại bỏ prefill context và dùng Speculative Decoding (MTP) tăng tốc decode.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Giảm số luồng decode từ 16 luồng (-t 16) về 4 luồng vật lý (-t 4)

```
before:  19.6 tok/s
after:   20.8 tok/s
speedup: 1.06x
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Khi chạy với `-t 16` (vượt quá 4 nhân vật lý và 8 luồng logical của chip i5-1135G7), các luồng ảo hyperthreading liên tục tranh chấp đơn vị thực thi (ALU/FPU) và bộ nhớ đệm L1/L2 của cùng một nhân vật lý. Việc oversubscribe này gây ra hiện tượng cache thrashing trong bộ đệm L3 và làm tăng vọt chi phí chuyển ngữ cảnh (context switching overhead) cùng độ trễ điều phối của hệ điều hành.

Khi điều chỉnh về `-t 4` (khớp hoàn hảo với 4 physical cores), mỗi luồng tính toán tensor được gán trọn vẹn tài nguyên của một nhân vật lý độc lập. Việc loại bỏ hoàn toàn chi phí tranh chấp tài nguyên và giảm tải cho bus bộ nhớ giúp nâng thông lượng sinh token từ 19.6 lên đỉnh 20.8 tok/s (tăng 1.06x), mang lại hiệu năng cao nhất và ổn định nhất trên kiến trúc phần cứng của máy.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** B2 (Batch-size sweep / chunked prefill) & B5/C8 (Semantic Caching simulation)

**Numbers:**

```
before:  101.6 tok/s (-b 256 -ub 256)
after:   136.5 tok/s (-b 128 -ub 128)
speedup: 1.34x
```

**Điều này nói lên gì mà deck chưa nói:**

1. **Chunked Prefill & Micro-batching (B2):** Micro-batch nhỏ (`-ub 128`) không chỉ giúp giảm TTFT jitter cho các decode request đang chờ trong hàng đợi mà còn cải thiện thông lượng prefill thực tế thêm **1.34x (101.6 -> 136.5 tok/s)** trên CPU/iGPU nhờ vừa vặn hơn với kích thước L2/L3 cache của từng core.
2. **Semantic Caching (C8):** Đạt hit rate 38% (3/8 request), giúp giảm độ trễ về **0 ms** và tiết kiệm 100% tài nguyên tính toán (bỏ qua hoàn toàn cả prefill và decode). Tuy nhiên, cache ngữ nghĩa cần cơ chế cô lập (salting per-tenant) để tránh rò rỉ dữ liệu xuyên người dùng qua kênh timing channel (theo công bố bảo mật NDSS'25).

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Bản quantization 2-bit (`UD-Q2_K_XL`) lại chạy chậm hơn 5.32x so với bản 4-bit (`Q4_K_M`) vì chi phí giải lượng tử hóa (dequantization) trên CPU lớn hơn nhiều so với lượng băng thông bộ nhớ tiết kiệm được.

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [x] 5 screenshots trong `submission/screenshots/`
- [x] `make verify` → **exit 0**
- [x] Repo GitHub ở chế độ **public**
- [x] Đã paste public URL vào VinUni LMS
- [x] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
