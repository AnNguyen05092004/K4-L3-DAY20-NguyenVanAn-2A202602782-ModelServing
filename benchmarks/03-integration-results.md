# 03 - Integrate: RAG pipeline run

Host `Darwin-arm64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 1425.3 | 1425.4 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 1239.4 | 1239.4 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 2710.1 | 2710.1 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **1791.6** · total **1791.6**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the context provided, **Goodput** is more useful than raw throughput because it focuses on **SLOs (Service Level Objects)** and **TTFT/TPOT targets** (throughput targets).

Specifically, the text states that Goodput counts only requests per second that met these targets, whereas **raw throughput** ignores SLOs and throughput at saturation. This means Goodput provides a more accurate and r

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation in GPU memory** caused by storing the key-value cache (KV cache) in non-contiguous pages.

By organizing the KV cache into non-contiguous pages, the model avoids the wasted space that would occur if all data were packed into a single contiguous block of memory. This allows the GPU to utilize more of its available memory, which is partic

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps to **avoid memory bandwidth bottlenecks** during the **prefill** phase.

Here is the breakdown based on the provided context:

*   **Prefill is compute-bound:** It requires significant processing power (CPU/GPU) to generate the model's weights and parameters.
*   **Decode is memory-bound:** It requires significant memory bandwidth to load and process the weights.


## Which N16-N19 pieces are real

Tôi chạy `pipeline.py` nguyên bản, **chưa nối code thật của N16–N19**. Khai báo cụ thể:

| Day | Mảnh | Trạng thái | Đang thay bằng gì |
|---|---|---|---|
| N16 Cloud/IaC | k8s / Compose | **stub** | chỉ chạy `localhost` trên laptop |
| N17 Data pipeline | Airflow DAG / batch job | **stub** | không có; dữ liệu là list Python trong file |
| N18 Lakehouse | Delta / Iceberg | **stub** | dict `TOY_DOCS` (6 câu) trong `pipeline.py` |
| N19 Vector + features | vector index + Feast | **stub** | keyword overlap (đếm từ trùng, từ dài > 3 ký tự), **không có embedding** |
| N20 Serving | `llama-server` | **real** | llama.cpp b10488, Qwen3.5 0.8B Q4_K_M, `--parallel 4` |

Hệ quả của việc stub: `embed = 0.0 ms` vì không có embedding server, và `retrieve = 0.0 ms`
(làm tròn 1 chữ số) vì chỉ duyệt 6 tài liệu trong RAM. Hai con số này **không phản ánh** chi
phí của một vector index thật; chúng chỉ là mốc "sàn".

**Dominant stage có đúng kỳ vọng không?** Có: `llm` chiếm 100% của total (1791.6 ms trung
bình; embed và retrieve đều làm tròn về 0.0). Tách thêm bằng `timings` của server (mean 3
query): prefill ≈ 110 ms (≈ 6% của llm), decode ≈ 1652 ms (≈ 92%), phần còn lại (~2%) là
overhead HTTP/xếp hàng. Decode chiếm áp đảo vì mỗi câu trả lời sinh 93–200 token ở
~77–85 tok/s, còn prompt chỉ 113–151 token nên prefill rất rẻ. Câu 3 chạm trần
`max_tokens=200` nên là query chậm nhất (2710 ms) và câu trả lời bị cắt giữa chừng.

**Muốn giảm một nửa latency thì tấn công stage nào?** Vẫn là `llm`, và cụ thể là **số token
decode**, không phải retrieval (đã ≈ 0 ms) hay prefill (≈ 110 ms). Đòn bẩy rẻ nhất là bắt
model trả lời ngắn (system prompt yêu cầu ≤ 2 câu, hạ `max_tokens`), vì latency gần như tỉ
lệ với số token sinh ra: giảm từ ~130 xuống ~65 token trung bình sẽ cắt gần một nửa. Đổi sang
quantization 2-bit chỉ nhanh hơn ~8% (xem `01-quickstart-results.md`), không đủ. Lưu ý: nếu
sau này thay stub bằng embedding server và vector index thật thì `embed`/`retrieve` sẽ không
còn bằng 0, và kết luận "llm chiếm 100%" có thể thay đổi; tôi chưa đo trường hợp đó.

**Một lưu ý về chất lượng retrieval:** với `k=3` và keyword overlap, câu 1 và 2 chỉ có 1
tài liệu khớp (điểm 1.0), hai tài liệu còn lại có điểm 0.0 nhưng vẫn bị nhét vào prompt làm
nhiễu. Câu 3 trả lời sai ý (nói splitting giúp tránh nghẽn băng thông ở *prefill*, trong khi
context nói prefill là compute-bound còn decode mới là memory-bound), một lỗi do model 0.8B
chứ không phải do latency.
