# Bonus - Batch-size sweep (chunked prefill)

Host `Windows-AMD64` · llama.cpp `b10488` ·
`threads=4` `ngl=99` · metric `pp512`

| -b (logical) | -ub (micro) | pp512 (tok/s) | vs best |
|:--|--:|--:|--:|
| 128 | 128 | 136.5 | 100% |
| 256 | 256 | 101.6 | 74% |
| 512 | 256 | 127.0 | 93% |
| 512 | 512 | 119.0 | 87% |
| 1024 | 512 | 126.6 | 93% |
| 2048 | 512 | 126.6 | 93% |

Best: `-b 128 -ub 128` at 136.5 tok/s
(1.34x the slowest point tested).

This sweep only measures the throughput half of the trade. The cost it hides is
TTFT for queued requests: a larger micro-batch holds the device longer per step,
so anything waiting behind it waits longer. To see both halves, re-run
`make load-50` with your best and worst settings via
`.venv/bin/python labs/02-serve/serve.py -- -b N -ub M` and compare P95.

## Your finding

Trong môi trường production có tải đồng thời cao, tôi sẽ chọn cấu hình `-b 512 -ub 256` hoặc `-b 128 -ub 128` (thay vì micro-batch quá lớn như `-ub 512`). 
Lý do: Micro-batch nhỏ giúp chia nhỏ các khối prompt prefill dài (chunked prefill), cho phép các bước decode của các user khác được xen kẽ (interleaving) liên tục mà không bị nghẽn (compute starvation).
Để đảm bảo cấu hình này không làm hỏng P95 latency khi server chịu tải cao (contended load), cần đo lường:
1. **TTFT P95 và P99** dưới tải bão hòa (ví dụ load test 50 users).
2. **Thời gian chờ trong hàng đợi (`requests_deferred` / queue time)** để đảm bảo việc chia nhỏ chunk không làm phình tổng thời gian prefill của chính request đó trong khi vẫn giữ tail latency ở mức thấp.
