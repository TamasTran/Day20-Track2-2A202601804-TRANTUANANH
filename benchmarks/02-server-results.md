# 02 - Serve: load test + saturation reading

Host `Linux-x86_64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=6` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 17 | 0.34 | 23000 | 29000 | 29000 | 7.2 | 0.0% |
| 50 | 27 | 0.50 | 23000 | 53000 | 53000 | 12.3 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.50x** (30% of linear) |
| P95 latency | **1.83x** |
| Effective concurrency at 50 users | 12.3 vs `--parallel 4` slots (occupancy/slot ratio 3.07) |

**Saturated.** Throughput delivered only 1.50x for 5x the offered load, and effective concurrency (12.3) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.50x while P95 moved 1.83x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

> **Small sample.** Only 17 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Your reading

Em thấy server bão hoà rõ ràng ở khoảng giữa 10 và 50 users. Bằng chứng thuyết phục nhất là con số **throughput chỉ tăng 1.50×** khi offered load tăng 5× (từ 10 lên 50 users). Nếu server chưa bão hoà, throughput phải scale gần tuyến tính — nhưng ở đây nó chỉ đi từ 0.34 RPS lên 0.50 RPS, nghĩa là server đã "hết chỗ" từ lâu.

Đồng thời, P95 tăng từ 29s lên 53s (1.83×) — tức P95 tăng nhanh hơn throughput. Phần latency thêm vào này là **queue time**, không phải compute time. Em biết điều này vì P50 hầu như không đổi (vẫn 23s ở cả hai mức load), nghĩa là thời gian server thực sự xử lý một request không thay đổi. Chỉ có những request xui xẻo rơi vào tail (P95, P99) mới bị chờ lâu hơn — đó là triệu chứng điển hình của queueing.

Effective concurrency ở 50 users là 12.3, trong khi server chỉ có 4 slot (`--parallel 4`). Occupancy/slot ratio = 3.07, tức trung bình mỗi slot đang "gánh" thêm 2 request chờ trong queue. Kết hợp với peak `n_busy_slots_per_decode` = 3.88/4 (từ `make metrics`), rõ ràng tất cả slot đã bận hết và request thừa chỉ có thể chờ.

Nếu em phải nâng goodput@SLO (ví dụ SLO = P95 < 35s), em sẽ tăng **`--parallel`** trước — vì bottleneck hiện tại là queue time, không phải compute. Thêm slot cho phép server xử lý đồng thời nhiều request hơn, giảm queue depth, từ đó kéo P95 xuống. Tăng thread count sẽ không giúp vì decode đã bị giới hạn bởi memory bandwidth (kết quả `make tune` cho thấy peak ở -t 3). Giảm `max_tokens` cũng là một option nhưng đó là trade-off về chất lượng output, không phải tuning thuần tuý.
