# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 15 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.80 of 4 slots (95%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 1951 |

Highest sampled value was **3.80 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Đỉnh batch width đo được từ gauge `/metrics` là **3.80 / 4 slots** (tương đương 95% công suất decode song song). Trong khi đó, `02-server-results.md` tính toán **Effective Concurrency = 11.4** dựa trên Định luật Little ($RPS \times Latency$). 
Sự khác biệt này là hoàn toàn hợp lý và nhất quán: `n_busy_slots_per_decode` (3.80) phản ánh số slot thực tế đang decode trong phần cứng (bị giới hạn cứng bởi `--parallel 4`), còn Effective Concurrency (11.4) tính cả các request đang nằm chờ trong hàng đợi (`requests_deferred` lên tới 46). Cả hai số liệu đều đáng tin cậy và phản ánh hai mặt: hiệu suất tận dụng slot decode và mức độ nghẽn hàng đợi.
