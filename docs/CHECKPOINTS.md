# Checkpoints — Day 20 Lab

Mỗi checkpoint gồm: **cần làm gì**, **sản phẩm**, **cần hiểu gì** và **cách tự kiểm
tra**. Lệnh chi tiết ở [docs/GUIDE.md](GUIDE.md); điểm ở [docs/RUBRIC.md](RUBRIC.md).

Windows: thay `make <target>` bằng `.\lab.ps1 <target>`.

---

## CP0 — Setup *(~20 phút · rubric 1, 2 · 10 điểm)*

**Cần làm**
- `make probe` → chọn cách chạy theo RAM (≥ 8 GB: Gemma 4 E2B · 4–8 GB:
  `LAB_MODEL=qwen35-0.8b` · < 4 GB: [cloud](CLOUD.md)).
- `make setup` (Windows: `bootstrap.ps1`) — tạo `.venv`, tải llama.cpp prebuilt, tải 2
  quantization của model.

**Sản phẩm**
- `hardware.json`, `models/active.json`
- Screenshot `01-hardware-probe.png`

**Cần hiểu**
- Vì sao lab dùng llama.cpp prebuilt thay vì vLLM / `llama-cpp-python`.
- Primary quant (4-bit) và compare quant (2-bit) khác nhau ở đâu.

**Tự kiểm tra**
- `cat models/active.json` có `model`, `primary_model`, `compare_model`.
- `make probe` in đúng số core physical/logical và RAM của máy bạn.

---

## CP1 — Đo baseline: TTFT / TPOT / percentile *(~20 phút · rubric 3, 4, 5 · 20 điểm)*

**Cần làm**
- `make bench` — đo 10 prompt cho mỗi quantization.
- Hỏi cùng một câu trên `make serve` và `serve.py --compare` để so chất lượng.
- Thay section "Your observation" trong report.

**Sản phẩm**
- `benchmarks/01-quickstart-results.md` (đã điền nhận xét)
- Screenshot `02-bench.png`

**Cần hiểu**
- TTFT = prefill (compute-bound); TPOT = decode (memory-bandwidth-bound).
- Vì sao ít bit hơn thường decode nhanh hơn — và khi nào **không** (máy compute-bound,
  chi phí dequantize).
- Vì sao báo P95/P99 chứ không chỉ trung bình.

**Tự kiểm tra**
- Bảng có **cả hai** quantization, TTFT và TPOT ở **hai cột riêng**.
- Bạn trả lời được: 2-bit nhanh hơn bao nhiêu, nhỏ hơn bao nhiêu, **có đáng không**.

---

## CP2 — Tune thread count *(~15 phút · số liệu cho rubric 11, chấm ở CP7)*

**Cần làm**
- `make tune` — sweep `-t` bằng `llama-bench`.
- Thay section "Your explanation" trong report.

**Sản phẩm**
- `benchmarks/01-tuning-tg128.md` (đã điền giải thích)

**Cần hiểu**
- Vì sao throughput thường đạt đỉnh quanh số core **physical** rồi đi ngang/giảm:
  thread thừa tranh cùng memory channel.
- Nếu curve của bạn khác kỳ vọng, cơ chế nào giải thích được.

**Tự kiểm tra**
- Chỉ ra được **knee** của curve và giải thích nó bằng một cơ chế cụ thể (bandwidth,
  cache, SMT, oversubscription), không chỉ chép lại con số.

---

## CP3 — Serve + smoke test *(~10 phút · rubric 6, 7 · 15 điểm)*

**Cần làm**
- Terminal 1: `make serve` (để chạy suốt các checkpoint sau).
- Terminal 2: `make smoke`.

**Sản phẩm**
- Screenshot `03-serve-and-smoke.png` — có **cả** server đang listen **và** output smoke.

**Cần hiểu**
- Endpoint OpenAI-compatible `/v1/chat/completions` và Prometheus `/metrics`.
- `--parallel` (số slot) và `--cont-batching` làm gì.

**Tự kiểm tra**
- Smoke in ra một câu trả lời thật và `llamacpp:tokens_predicted_total` **khác 0**.

---

## CP4 — Load test + continuous batching *(~15 phút · rubric 8, 9 · 10 điểm)*

**Cần làm**
- `make load-10` (10 users, 60 s).
- `make load-50` (50 users, 60 s) và **trong lúc nó đang chạy**, terminal 3:
  `make metrics`.
- Thay section "Your observation" trong report batching.

**Sản phẩm**
- `benchmarks/locust-10_stats.csv`, `benchmarks/locust-50_stats.csv`
- `benchmarks/02-server-batching-u50.md` + `02-server-metrics-u50.csv`
- Screenshot `04-locust-10.png`, `05-locust-50.png`

**Cần hiểu**
- `n_busy_slots_per_decode` tiến gần `--parallel` = continuous batching đang gộp request
  vào chung decode step.
- `requests_deferred` > 0 nghĩa là có request phải xếp hàng.

**Tự kiểm tra**
- Peak `n_busy_slots_per_decode` **rõ ràng lớn hơn 1**. Nếu ≈ 1, bạn đã chạy `metrics`
  khi server rảnh — chạy lại, chồng thời gian với `load-50`.
- Screenshot locust thấy dòng `# reqs · Median · 95%ile · 99%ile`.

---

## CP5 — Saturation reading *(~15 phút · rubric 10 · 10 điểm)*

**Cần làm**
- `make load-report`.
- Thay section "Your reading" trong report.

**Sản phẩm**
- `benchmarks/02-server-results.md` (đã điền phân tích)

**Cần hiểu**
- Little's Law: effective concurrency = RPS × latency trung bình. Lớn hơn số slot →
  phần P95 tăng thêm là **queue time**, không phải compute time.
- Goodput@SLO khác peak throughput ở đâu.

**Tự kiểm tra**
- Bạn nêu được: RPS tăng bao nhiêu lần, P95 tăng bao nhiêu lần, effective concurrency
  so với số slot, và knob nào bạn sẽ đổi **trước** để nâng goodput — kèm lý do.

---

## CP6 — RAG pipeline *(~15 phút · rubric 12, 13 · 15 điểm)*

**Cần làm**
- `make pipeline` (server vẫn đang chạy).
- (Tuỳ chọn) thay `STUB 1` / `STUB 2` trong `labs/03-integrate/pipeline.py` bằng code
  N16–N19 của bạn.
- Thay section "Which N16-N19 pieces are real" trong report.

**Sản phẩm**
- `benchmarks/03-integration-results.md` (đã điền)

**Cần hiểu**
- Latency của pipeline nằm ở stage nào (embed / retrieve / llm) và vì sao.
- Prefix caching: vì sao system prompt phải giống nhau từng byte giữa các lần gọi.

**Tự kiểm tra**
- Cả 3 query chạy hết và in ra context đã retrieve.
- Bạn khai báo đúng từng mảnh N16–N19 là **real** hay **stub** (stub không mất điểm,
  khai sai mới mất).

---

## CP7 — REFLECTION + verify *(~30 phút · rubric 11, 14 · 20 điểm)*

**Cần làm**
- Điền [`submission/REFLECTION.md`](../submission/REFLECTION.md) §1–§5 (§6–§9 tuỳ chọn).
- Đủ 5 screenshot trong `submission/screenshots/`.
- `make verify`.

**Sản phẩm**
- `submission/REFLECTION.md` hoàn chỉnh, 5 screenshot, `make verify` exit 0.

**Cần hiểu**
- §5 "The single change that mattered most": before/after thật + **cơ chế** giải thích
  vì sao nó nhanh hơn. Đây là phần grader đọc kỹ nhất.

**Tự kiểm tra**
- `make verify` → **exit 0**.
- Số trong REFLECTION khớp với `benchmarks/*.md`.

---

## CP8 — Nộp bài *(~5 phút)*

**Cần làm**
- Push lên repo public tên `K4-L3-DAY20-HoVaTen-MSSV-ModelServing`, paste URL vào LMS.

**Sản phẩm**
- URL repo public trên LMS **trước 23:59 (UTC+7) ngày làm lab**.

**Tự kiểm tra**
- Checklist ở [docs/SUBMISSION.md §6](SUBMISSION.md#6-kiểm-tra-trước-khi-nộp).

---

## CP-Bonus — *(optional · tối đa +10 điểm · chỉ sau khi CP7 xong)*

**Cần làm**
- Chọn 1–2 mục B1–B5 trong [docs/bonus/README.md](bonus/README.md).

**Sản phẩm**
- `benchmarks/bonus-*.md` (đã điền), REFLECTION §6.

**Cần hiểu**
- Một finding giải thích sâu có giá trị hơn năm bảng số nông.

**Tự kiểm tra**
- §6 có before/after **từ bonus track** (không dùng lại kết quả `make tune`).
