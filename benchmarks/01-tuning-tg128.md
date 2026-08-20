# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **4 physical · 8 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 19.9 | 96% |
| 2 | 19.8 | 95% |
| 4 | 20.8 | 100% |
| 8 | 19.8 | 95% |
| 16 | 19.6 | 95% |

**Best**: `-t 4` at 20.8 tok/s
**Slowest tested**: `-t 16` at 19.6 tok/s (1.06x spread)
**Against the physical-core default** (`-t 4`, 20.8 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=4 make bench
```

## Your explanation

Điểm uốn tối ưu (knee point) nằm chính xác tại số nhân vật lý của CPU (`-t 4` đạt đỉnh 20.8 tok/s). Khi tăng số luồng lên mức logical cores (`-t 8` - 19.8 tok/s) hoặc vượt mức vật lý (`-t 16` - 19.6 tok/s), thông lượng bị giảm do:
1. **Tranh chấp tài nguyên luồng (SMT/Hyperthreading overhead):** Các luồng ảo chia sẻ chung đơn vị thực thi (ALU/FPU) và bộ nhớ đệm L1/L2 của cùng một nhân vật lý.
2. **Xung đột bộ nhớ và Cache Thrashing:** Quá nhiều luồng tranh chấp bus bộ nhớ và gây hiện tượng thrashing trong L3 cache, tăng chi phí đồng bộ hóa (thread sync / context switching).
Do đó, cấu hình `-t 4` (bằng đúng số physical cores) mang lại hiệu quả cao nhất mà không bị lãng phí chi phí điều phối.
