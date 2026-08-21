# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Trần Tuấn Anh
**Cohort:** A20-K2
**Ngày submit:** 2026-08-21

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Linux (WSL2 trên Windows) — kernel `6.18.33.2-microsoft-standard-WSL2`
- **CPU:** Intel(R) Core(TM) i7-8850H CPU @ 2.60GHz
- **Cores:** 6 physical / 12 logical
- **CPU extensions:** AVX2 (có) · AVX-512 (không) · NEON (không)
- **RAM:** 7.6 GB
- **Accelerator:** NVIDIA Quadro P1000, 4096 MiB (CUDA backend detected)
- **llama.cpp asset đã tải:** llama.cpp `b10488` (prebuilt binary, không compile)
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** `Qwen3.5-0.8B-Q4_K_M.gguf` (primary) + `Qwen3.5-0.8B-UD-Q2_K_XL.gguf` (compare)

**Chạy ở đâu:** Laptop của em (Dell Precision, chạy WSL2 trên Windows). RAM chỉ có 7.6 GB nên hệ thống tự recommend Qwen3.5 0.8B thay vì Gemma 4 E2B — em theo đúng recommendation đó.
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Ban đầu em định chạy native Windows nhưng phát hiện lab dùng Makefile nên chuyển sang WSL2. Em chạy hoàn toàn bằng các lệnh `make` (như `make setup`, `make bench`, v.v.) trực tiếp trên môi trường WSL2. Model Qwen3.5 0.8B tải nhanh (~0.9 GB tổng cộng), không bị chặn bởi mạng trường. Không có bước nào fail cần workaround đặc biệt.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 6084 | 350 / 544 | 59.1 / 77.4 | 3766 / 5422 / 5422 | 16.9 |
| UD-Q2_K_XL | 0.39 | 16355 | 475 / 1035 | 47.3 / 66.6 | 3455 / 4253 / 4253 | 21.1 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

Bản 2-bit decode nhanh hơn 1.25× (21.1 vs 16.9 tok/s) và nhẹ hơn 22% trên disk. Nhưng TTFT P95 của nó bất ổn hơn nhiều (1035 ms vs 544 ms). Khi em thử hỏi cùng câu trên cả hai, bản Q4 trả lời mạch lạc hơn — bản Q2 đôi chỗ bị lặp ý. Với model 0.8B vốn đã nhỏ, quantize xuống 2-bit làm mất thêm precision đáng kể. Em sẽ chọn Q4 cho RAG pipeline vì cần output đáng tin, chấp nhận chậm hơn một chút.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.34 | 23000 | 29000 | 29000 | 7.2 | 0.0% |
| 50 | 0.50 | 23000 | 53000 | 53000 | 12.3 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.50×
- **P95 tăng:** 1.83×
- **Effective concurrency ở 50 users:** 12.3 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.88 / 4 slots

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Em thấy server bão hoà rõ ở khoảng giữa 10–50 users. Bằng chứng rõ nhất: throughput chỉ tăng 1.50× cho 5× offered load — server không còn capacity để chuyển thêm request thành throughput. P95 nhảy từ 29s lên 53s trong khi P50 không đổi (23s), chứng tỏ phần latency tăng thêm hoàn toàn là **queue time** — request chờ slot trống. Em biết đó không phải compute time vì decode speed không thay đổi (P50 giữ nguyên). Effective concurrency 12.3 trên chỉ 4 slot → occupancy ratio 3.07 → mỗi slot có ~2 request đang xếp hàng.
Nếu cần nâng goodput@SLO (giả sử SLO = P95 < 35s), em sẽ tăng `--parallel` trước — vì bottleneck là queue time, không phải compute. Tăng thread count không giúp (decode bị chặn bởi memory bandwidth, peak ở -t 3 theo `make tune`).

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | stub | Chạy local trên WSL2, không có cloud infra |
| N17 Data pipeline | stub | Không có ETL, data hardcode trong script |
| N18 Lakehouse | stub | Không có data lake hay storage layer |
| N19 Vector + features | stub | Dùng keyword overlap, không có embedding model |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 10444.5 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

Bottleneck hoàn toàn nằm ở LLM stage — em thấy embed và retrieve gần như 0 ms vì cả hai đều là stub (keyword matching, không có embedding model thật). Điều này khớp với kỳ vọng: khi retrieval là stub thì LLM inference chiếm toàn bộ. Nếu phải giảm latency 2×, em sẽ giảm `max_tokens` (hiện tại model generate 82–200 token/query) hoặc dùng bản Q2 nhanh hơn (21.1 vs 16.9 tok/s). Prompt caching cũng sẽ giúp nếu nhiều query share context prefix.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Hạ `-t` (thread count) từ 6 (mặc định = physical core count) xuống 3

```
before:  9.2 tok/s  (tg128 với -t 6)
after:   11.8 tok/s (tg128 với -t 3)
speedup: 1.28×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

_Giải thích như đang nói với bạn ngồi cạnh. Bám vào **cơ chế**, không phải "vibes":
memory bandwidth? vector width? cache residency? scheduling? queueing? Nếu kết quả
**khác** với kỳ vọng từ deck — nói rõ, và giải thích vì sao. Grader thưởng điểm cho
lập luận đúng về một kết quả bất ngờ, hơn là một con số đẹp không được giải thích._

Em ban đầu nghĩ rằng dùng hết 6 physical cores sẽ cho tốc độ tốt nhất — nhiều core hơn = nhiều phép tính song song hơn. Nhưng thực tế hoàn toàn ngược lại. Lý do nằm ở bản chất của decode phase trong LLM inference: nó bị giới hạn bởi **memory bandwidth**, không phải compute.

Mỗi token decode cần đọc toàn bộ model weights từ RAM một lần (Qwen3.5 0.8B Q4_K_M ≈ 0.5 GB). Trên máy em (Intel i7-8850H, DDR4), băng thông bộ nhớ có giới hạn cố định — khoảng 34–38 GB/s theo spec. Khi dùng 3 thread, lượng data mỗi thread cần fetch vừa đủ để saturate memory bus mà không gây tranh chấp. Nhưng khi tăng lên 6 thread, **tất cả 6 core đều cạnh tranh cùng một memory bus** — mỗi core phát ra memory request nhưng tổng bandwidth không tăng. Kết quả: mỗi thread đợi lâu hơn, cache line bị invalidate nhiều hơn, và throughput tổng thể giảm.

Điều thú vị nhất là kết quả ở -t 12 (logical cores, tức Hyper-Threading): chỉ 0.8 tok/s — chậm hơn **15× so với peak**. Hyper-Threading chia sẻ L1/L2 cache và execution units trên cùng physical core, gây cache thrashing nghiêm trọng cho workload memory-bound. Data của thread này liên tục bị thread kia đẩy ra khỏi cache, khiến gần như mọi access đều miss và phải đi ra RAM. Đây là minh hoạ rõ nhất cho việc Hyper-Threading có thể **hại** performance khi workload là memory-bound thay vì compute-bound.

**Tóm lại**: bottleneck nằm ở memory bandwidth, và chỉ cần vừa đủ thread để saturate bus là tối ưu. Thêm thread quá mức gây tranh chấp bandwidth và cache thrashing, phản tác dụng hoàn toàn.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

_(Em không làm phần bonus.)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

Điều làm em ngạc nhiên nhất là peak performance nằm ở -t 3 chứ không phải -t 6. Em luôn nghĩ "dùng hết core = nhanh nhất", nhưng lab này cho thấy rõ rằng với workload memory-bound, thêm thread quá mức không chỉ không giúp mà còn làm tệ hơn gấp nhiều lần. Đặc biệt, -t 12 (Hyper-Threading) chậm hơn 15× so với -t 3 là một con số em không ngờ tới — nó khiến em thực sự hiểu tại sao auto-tuning thread count lại quan trọng thay vì dùng mặc định.

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
      đã được thay bằng nhận xét của em
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [ ] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
