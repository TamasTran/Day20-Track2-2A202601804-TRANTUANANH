# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Linux-x86_64` · llama.cpp `b10488`
Settings: `threads=6` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 6084 | 350 / 544 | 59.1 / 77.4 | 3766 / 5422 / 5422 | 16.9 |
| UD-Q2_K_XL | 0.39 | 16355 | 475 / 1035 | 47.3 / 66.6 | 3455 / 4253 / 4253 | 21.1 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.25x faster** than `Q4_K_M` here, for 0.11 GB less on disk.

## Your observation

Em thấy bản 2-bit (UD-Q2_K_XL) decode nhanh hơn rõ rệt: 21.1 tok/s so với 16.9 tok/s của Q4_K_M — tức nhanh hơn khoảng 1.25×. Dung lượng cũng nhẹ hơn: 0.39 GB so với 0.50 GB, tiết kiệm được ~22% disk. Nhưng cái đánh đổi nằm ở chỗ khác: TTFT P50 của bản 2-bit là 475 ms so với 350 ms (chậm hơn 36%), và TTFT P95 thì chênh lệch lớn hơn nhiều — 1035 ms so với 544 ms. Điều này cho thấy prefill ở bản 2-bit không ổn định bằng.

Khi em thử hỏi cả hai cùng một câu ("Giải thích PagedAttention"), bản Q4 trả lời mạch lạc và đầy đủ hơn, còn bản Q2 đôi chỗ bị lặp ý hoặc câu văn hơi rời rạc. Với model nhỏ 0.8B thì bản thân chất lượng đã không cao, nên việc quantize xuống 2-bit làm nó tệ thêm một bậc nữa là điều dễ hiểu — fewer bits per weight nghĩa là mất thêm precision.

**Kết luận:** Trên máy em, bản 2-bit đáng dùng nếu chỉ cần throughput cao và chấp nhận câu trả lời "vừa đủ" (ví dụ summarize ngắn, chatbot đơn giản). Nhưng nếu cần câu trả lời có chất lượng tối thiểu cho RAG hay QA, em sẽ chọn Q4_K_M — chậm hơn một chút nhưng output đáng tin hơn.
