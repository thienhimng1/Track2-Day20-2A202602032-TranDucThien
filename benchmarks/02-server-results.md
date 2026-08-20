# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=4` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 12 | 0.23 | 34000 | 43000 | 43000 | 7.2 | 0.0% |
| 50 | 16 | 0.31 | 41000 | 52000 | 52000 | 11.4 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.35x** (27% of linear) |
| P95 latency | **1.21x** |
| Effective concurrency at 50 users | 11.4 vs `--parallel 4` slots (occupancy/slot ratio 2.86) |

**Saturated.** Throughput delivered only 1.35x for 5x the offered load, and effective concurrency (11.4) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

P95 grew no faster than throughput (1.21x vs 1.35x), so this server still has headroom at 50 users.

> **Small sample.** Only 12 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Your reading

Server đạt điểm bão hòa (saturation point) ở khoảng giữa 10 và 50 users. 
Bằng chứng thuyết phục nhất là:
1. **Thông lượng tăng phi tuyến tính:** Khi tải tăng gấp 5 lần (10 lên 50 users), RPS chỉ tăng từ 0.23 lên 0.31 (chỉ tăng **1.35x**, tương đương 27% so với mức tuyến tính kỳ vọng).
2. **Hàng đợi phình to (Queue Congestion):** Effective concurrency ở 50 users đạt **11.4** (vượt xa 4 slots khả dụng, tỷ lệ occupancy/slot = 2.86), đồng thời `requests_deferred` đo được là 46. Tải tăng thêm hoàn toàn bị chuyển thành thời gian chờ hàng đợi (queue time).

Để nâng cao **goodput@SLO**, tôi sẽ thay đổi knob `--parallel` (tăng số slot xử lý đồng thời từ 4 lên 8) đầu tiên vì đây là nút thắt trực tiếp giới hạn năng lực batching của engine khi GPU/CPU vẫn còn dung lượng VRAM/RAM để chứa thêm KV cache slots.
