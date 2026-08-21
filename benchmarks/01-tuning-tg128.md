# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Linux-x86_64` · llama.cpp `b10488`
CPU: **6 physical · 12 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 9.0 | 76% |
| 3 | 11.8 | 100% |
| 6 | 9.2 | 78% |
| 12 | 0.8 | 7% |
| 24 | 1.6 | 14% |

**Best**: `-t 3` at 11.8 tok/s
**Slowest tested**: `-t 12` at 0.8 tok/s (15.13x spread)
**Against the physical-core default** (`-t 6`, 9.2 tok/s): 1.28x

Use this in your run:

```bash
LAB_N_THREADS=3 make bench
```

## Your explanation

Em thấy điểm knee nằm ở **-t 3** (11.8 tok/s), không phải ở 6 physical cores như em kỳ vọng ban đầu. Khi tăng lên -t 6 thì tốc độ giảm xuống 9.2 tok/s (chỉ còn 78% so với best), và đến -t 12 thì rơi thảm hại — 0.8 tok/s, tức chậm hơn 15× so với peak.

Lý do em nghĩ là thế này: decode phase của LLM inference bị giới hạn bởi **memory bandwidth**, không phải compute. Mỗi token cần đọc toàn bộ model weights từ RAM một lần. Máy em dùng Intel i7-8850H với 7.6 GB RAM DDR4 — băng thông bộ nhớ của nó có giới hạn cố định. Khi dùng 3 thread, lượng data mỗi thread cần fetch vừa đủ để giữ memory bus bận mà không bị tranh chấp. Nhưng khi tăng lên 6 thread, các thread bắt đầu cạnh tranh nhau memory bandwidth — mỗi thread cần đọc weights nhưng tổng bandwidth không tăng theo, nên thực tế mỗi thread chờ lâu hơn.

Đến -t 12 (bằng logical cores, tức tính cả Hyper-Threading), tình hình tệ hơn rất nhiều: 2 thread trên cùng 1 physical core chia sẻ L1/L2 cache và execution units, gây **cache thrashing** — data của thread này bị thread kia đẩy ra khỏi cache liên tục. Với workload memory-bound như decode, Hyper-Threading không giúp gì mà còn hại thêm vì tăng cache miss rate. Kết quả -t 24 (4× physical) nhỉnh hơn -t 12 một chút (1.6 tok/s), có thể do OS scheduler phân phối thread đều hơn ở mức over-subscribe cao, nhưng vẫn rất chậm.

**Tóm lại:** bottleneck là memory bandwidth, nên chỉ cần vừa đủ thread để saturate bus là tối ưu. Trên máy em, con số đó là 3 — bằng đúng nửa physical core count.
