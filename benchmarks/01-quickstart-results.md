# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=4` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 15879 | 710 / 866 | 50.7 / 59.8 | 3902 / 4518 / 4518 | 19.7 |
| UD-Q2_K_XL | 0.39 | 11980 | 1159 / 1410 | 271.0 / 300.6 | 18273 / 20134 / 20134 | 3.7 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **5.32x SLOWER** than `Q4_K_M` here, despite being 0.11 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

Trên máy của tôi, bản 2-bit (`UD-Q2_K_XL`) giải mã chậm hơn 5.32x so với bản 4-bit (`Q4_K_M`) (3.7 tok/s vs 19.7 tok/s; TPOT P50 tăng từ 50.7ms lên 271.0ms), dù chỉ tiết kiệm được 0.11 GB RAM (0.39 GB vs 0.50 GB). Nguyên nhân do chi phí tính toán giải lượng tử hóa (dequantization overhead) của định dạng 2-bit trên CPU/iGPU lớn hơn nhiều so với lượng băng thông bộ nhớ tiết kiệm được. Do đó, bản 2-bit hoàn toàn **không đáng dùng** trên cấu hình máy này so với bản 4-bit.
