# 03 - Integrate: RAG pipeline run

Host `Linux-x86_64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 16729.5 | 16729.6 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 6531.5 | 6531.7 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 8072.5 | 8072.7 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **10444.5** · total **10444.7**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the context provided, **Goodput** is more useful than raw throughput because it focuses on the specific metrics that matter for system stability and performance:

1.  **TTFT (Throughput to Target Frequency):** Goodput counts only requests per second that met the Target Throughput and Target Processing Time (TTFT) targets. This ensures that the system never runs below its intended performa

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation** in GPU memory.

By storing the Key-Value (KV) cache in non-contiguous pages, the model avoids the wasted space that would exist if the KV cache were stored contiguously (e.g., in a standard contiguous block of memory). This non-contiguous layout allows for more efficient memory usage and better cache locality on the GPU.

**When does splitting prefill and decode help?**

> Based on the provided context, splitting prefill and decode helps primarily to **avoid redundant computation**.

The context explains that prefill is compute-bound, meaning it requires significant processing power, while decode is memory-bandwidth-bound. By splitting them into separate pools, the system can skip the expensive prefill step entirely for tokens that have already been processed in a s


## Which N16-N19 pieces are real

- **N16 (Cloud/IaC):** stub — em chạy local trên WSL2, không dùng cloud infrastructure nào.
- **N17 (Data pipeline):** stub — không có ETL hay data ingestion pipeline thực tế, dữ liệu được hardcode sẵn trong script.
- **N18 (Lakehouse):** stub — không có data lake hay storage layer nào, context documents nằm ngay trong code.
- **N19 (Vector + features):** stub — retrieval dùng **keyword overlap** (so từ khóa trùng), không có vector embedding hay feature store. Bằng chứng: output ghi `retrieval backend: keyword overlap` và `embed: 0.0 ms` — nếu có embedding model thật thì không thể 0 ms.
- **N20 (Serving):** real — llama-server chạy Qwen3.5 0.8B thật, trả lời các câu hỏi với nội dung có logic.

**Về dominant stage:** Em thấy LLM chiếm **100% total latency** (mean 10444.5 ms / 10444.7 ms). Điều này hoàn toàn khớp với kỳ vọng vì embed và retrieve đều là stub (keyword matching ≈ 0 ms). Ngay cả khi có embedding model thật, em nghĩ LLM vẫn sẽ chiếm phần lớn — nhưng tỉ lệ sẽ giảm xuống ~95% thay vì 100%, vì embedding thường mất vài chục ms cho mỗi query.

**Nếu phải giảm latency pipeline 2×**, em sẽ tấn công vào **LLM stage** — vì nó chiếm gần như toàn bộ thời gian. Cụ thể: giảm `max_tokens` (hiện tại model generate 82–200 token mỗi query, khá dài), hoặc dùng quantization nhỏ hơn (UD-Q2_K_XL decode 21.1 tok/s thay vì 16.9 tok/s). Prompt caching cũng giúp nếu nhiều query chia sẻ cùng context prefix — lúc đó prefill sẽ được skip phần lớn.
