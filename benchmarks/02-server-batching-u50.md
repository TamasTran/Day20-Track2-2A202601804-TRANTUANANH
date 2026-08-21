# 02 - Continuous batching under load (u50)

Host `Linux-x86_64` · `--parallel 4` · 29 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.88 of 4 slots (97%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 2620 |

Highest sampled value was **3.88 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Em thấy peak `n_busy_slots_per_decode` đạt **3.88 / 4 slots** (97%), nghĩa là gần như toàn bộ 4 slot đều bận đồng thời trong phần lớn thời gian decode. Con số này chứng minh continuous batching đang thực sự hoạt động — server không phải xử lý request lần lượt mà đang pack nhiều request vào chung một decode step.

Điều thú vị là `requests_deferred` lên tới 46 — tức là có 46 request phải chờ vì 4 slot đã đầy. Đây chính là nguồn gốc của queue time mà em thấy phản ánh trong P95 latency ở file `02-server-results.md` (P95 từ 29s nhảy lên 53s khi tăng từ 10 lên 50 users). Latency tăng thêm không phải vì server compute chậm hơn — server vẫn decode ở cùng tốc độ — mà vì request mới phải xếp hàng chờ slot trống.

So sánh với effective concurrency trong `02-server-results.md` (12.3 ở 50 users): con số 3.88 slots busy là hợp lý vì effective concurrency tính theo Little's Law bao gồm cả request đang queued, trong khi `n_busy_slots_per_decode` chỉ đếm request đang thực sự được decode. 12.3 request in-flight nhưng chỉ 4 slot → ~8 request đang chờ trong queue ở bất kỳ thời điểm nào, khớp với `requests_deferred` cao.
