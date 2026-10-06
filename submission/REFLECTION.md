# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyễn Văn An
**MSSV:** 2A202602782
**Cohort:** K4-L3
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** macOS 26.5.2 (Darwin 25.5.0, arm64)
- **CPU:** Apple M4
- **Cores:** 10 physical / 10 logical
- **CPU extensions:** NEON
- **RAM:** 16 GB
- **Accelerator:** Apple Metal (có sẵn trong binary prebuilt, `ngl=99`)
- **llama.cpp asset đã tải:** llama-b10488-bin-macos-arm64.tar.gz
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi (không dùng Colab/Kaggle)

**Setup story** (≤ 80 chữ): Ổ đĩa chỉ còn ~3.3 GB trống nên không đủ chỗ cho Gemma 4 E2B (~5.2 GB); tôi dùng Qwen3.5 0.8B (~0.9 GB). `make setup` lần đầu fail vì `CERTIFICATE_VERIFY_FAILED` (Python 3.13 cài từ python.org trên macOS thiếu CA bundle); tôi workaround bằng `SSL_CERT_FILE` trỏ tới `certifi` trong venv rồi chạy lại `setup.py`.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 1075 | 56 / 62 | 10.5 / 10.8 | 702 / 739 / 739 | 95.4 |
| UD-Q2_K_XL | 0.39 | 1027 | 54 / 63 | 9.7 / 10.7 | 662 / 732 / 732 | 103.4 |

**Quan sát** (≤ 60 chữ): 2-bit chỉ nhanh hơn 1.08× (TPOT 9.7 so với 10.5 ms) dù file nhỏ hơn 22%, và tôi không thấy lợi thế chất lượng: hỏi cùng bộ câu hỏi trên cả hai (temperature 0), cả hai đều bịa ở câu khó, bản 2-bit ra 381 cho 17×23 khi trình bày từng bước. **Không đáng** khi RAM 16 GB; chọn Q4_K_M.

Ghi chú: lần `make bench` đầu tiên (cache lạnh) cho Q4_K_M load 2076 ms và TTFT P95 203 ms; bảng trên là lần chạy thứ hai (cache ấm). Chi tiết ở `benchmarks/01-quickstart-results.md`.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 1.87 | 3600 | 5900 | 6500 | 7.0 | 0 |
| 50 | 1.93 | 24000 | 27000 | 28000 | 38.2 | 0 |

- **Offered load tăng 5×, throughput thực tăng:** 1.03×
- **P95 tăng:** 4.58×
- **Effective concurrency ở 50 users:** 38.2 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.96 / 4 slots

**Saturation reading** (≤ 80 chữ): Server bão hoà từ ≤ 10 users: effective concurrency ở 10 users đã là 7.0 > 4 slot, và RPS gần như phẳng (1.87 → 1.93) khi tải ×5 trong lúc P95 ×4.58. Phần tăng thêm là queue time: một request đơn chỉ ~0.5–1 s nhưng ở 50 users trung bình 19.8 s, và server báo ~44 request `deferred`. Tôi đổi trước **độ dài output** (`max_tokens`), không phải `--parallel`: thí nghiệm của tôi cho thấy 4→8 slot chỉ tăng RPS 1.06× vì tổng throughput đã chạm trần ~105–111 tok/s.

Ghi chú về screenshot: ảnh `04-locust-10.png` và `05-locust-50.png` là ảnh chụp log console của chính lần chạy `make load-10` / `make load-50` đã lưu lại (`benchmarks/locust-10-console.log`, `locust-50-console.log`), không chụp lúc locust đang chạy. Tổng cuối trong log (100 và 115 request) lệch nhẹ so với `02-server-results.md` (98 và 114 request) vì locust in tổng cuối sau khi file CSV đã được ghi.

Số liệu thí nghiệm `--parallel` 1/4/8 nằm ở `benchmarks/02-parallel-experiment.md` (không phải target `make`; có một lần chạy `--parallel 8` bị loại vì mỗi slot chỉ còn 256 token ctx nên request dài bị lỗi 400, lý do ghi trong file đó).

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | k8s / Compose | stub (chỉ chạy localhost) |
| N17 Data pipeline | Airflow DAG / batch job | stub (không có, dữ liệu là list Python) |
| N18 Lakehouse | Delta / Iceberg | stub (dict `TOY_DOCS` 6 câu trong `pipeline.py`) |
| N19 Vector + features | vector index + Feast | stub (keyword overlap, không có embedding) |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 1791.6 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): Đúng kỳ vọng: llm chiếm ~100%, trong đó decode ≈ 92% và prefill ≈ 6% (mỗi câu trả lời 93–200 token). Embed/retrieve = 0 vì là stub, nên không phản ánh vector index thật. Muốn giảm 2× tôi tấn công **số token decode** (system prompt bắt trả lời ≤ 2 câu, hạ `max_tokens`), không phải retrieval hay quantization (chỉ ~8%).

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** hạ số thread `-t` từ 10 (mặc định = số core vật lý) xuống 1, với Metal offload (`-ngl 99`)

```
before:  99.2 tok/s  (-t 10, tg128, make tune)
after:   104.1 tok/s (-t 1,  tg128, make tune)
speedup: 1.05×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Kết quả này **không giống kỳ vọng từ deck** (tăng tới số core rồi phẳng/giảm): đường cong của tôi phẳng từ 1 đến 5 thread (104.1 → 103.0 tok/s), giảm nhẹ ở 10 (99.2) và giảm rõ ở 20 thread (81.5, −22%), nên peak nằm ở `-t 1` chứ không ở số core. Lý do là với `-ngl 99` toàn bộ layer chạy trên GPU Metal: decode (đọc weight, nhân ma trận) do GPU làm, thread CPU chỉ dựng và gửi command buffer. Thêm thread không thêm công việc hữu ích mà chỉ thêm chi phí đồng bộ và tranh lịch, nên nhiều thread hơn thì chậm hơn. Cần nói thẳng: 1.05× nhỏ so với độ lệch chuẩn của llama-bench (~3–4 tok/s), nên riêng một lần đo không đủ kết luận. Tôi lặp lại ngl=99 với 6 rep, hai lượt: `-t 10` luôn thấp hơn `-t 1`/`-t 4` khoảng 15% (70.5 / 72.6 so với 83.8 / 84.7 tok/s), nên *thứ tự* nhất quán, dù số tuyệt đối thấp hơn lần `make tune` (tôi nghi do máy nóng nhưng chưa đo nhiệt độ).

Để kiểm chứng giả thuyết "thread chỉ quan trọng khi tính toán chạy trên CPU", tôi chạy thêm CPU thuần (`-ngl 0`, 3 rep, chạy `llama-bench` tay, log ở `benchmarks/01-tuning-extra-raw.txt`): `-t 1` 57.0 · `-t 2` 95.2 · `-t 4` 100.0 · **`-t 6` 104.2** · `-t 8` 94.2 · `-t 10` 57.7 tok/s. Đây mới là đường cong kỳ vọng: tăng mạnh lên 2 thread (1.67×), gần như phẳng ở ~100 tok/s (≈ 50 GB/s, dấu hiệu chạm trần băng thông mà cụm CPU kéo được, nên decode bị chặn bởi memory bandwidth chứ không phải FLOPs), rồi sụp ở 10 thread. Tôi giải thích sụp ở 10 thread bằng việc M4 gồm 4 core P + 6 core E và mỗi op kết thúc bằng một barrier, nên tốc độ bị quyết định bởi thread chậm nhất; đây là suy luận từ cấu trúc core, tôi chưa ghim thread vào P-core để xác nhận. Điểm `-t 20` CPU thuần tôi dừng sau hơn 23 phút không có kết quả (các điểm khác mất vài giây). Một phát hiện đi kèm: ở decode với model 0.8B, CPU thuần 6 thread (104.2 tok/s) **ngang** Metal (104.1 tok/s), nên tôi không có bằng chứng GPU làm decode nhanh hơn; lợi thế GPU, nếu có, nằm ở prefill mà tôi chưa đo.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** không làm bonus track. (Thí nghiệm `--parallel` 1/4/8 là phân tích bổ sung cho §3, không phải B1–B5.)

**Numbers:**

```
before:  <số>
after:   <số>
speedup: <X.Y>×
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Hai điều. (1) Tổng throughput của server chỉ ~105–111 tok/s ở cả `--parallel 4` và `8`, xấp xỉ tốc độ decode của một luồng đơn (~95–104 tok/s), nên continuous batching tuy "bận 3.96/4 slot" lại chỉ cho lợi ích rõ khi đi từ 1 lên 4 slot (RPS 1.23 → 1.85). (2) `--parallel` chia đều `ctx-size` cho các slot: ở `--parallel 8` mỗi slot còn 256 token và request dài bị từ chối với lỗi 400, làm RPS nhìn đẹp giả tạo (3.2) cho tới khi tôi đọc log server.

---

## 8. Self-check trước khi push

- [ ] `hardware.json` committed
- [ ] `models/active.json` committed
- [ ] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [ ] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [ ] `benchmarks/02-server-results.md` committed (`make load-report`)
- [ ] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [ ] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [ ] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [ ] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [ ] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Claude Code (Claude Sonnet 5.5) được dùng để: chạy các lệnh của lab trên máy tôi (`make probe/setup/bench/tune/serve/smoke/load-*/metrics/load-report/pipeline`), chạy các thí nghiệm bổ sung (so chất lượng 2 quantization, `llama-bench` CPU thuần, `--parallel` 1/4/8), và soạn nháp các đoạn nhận xét trong `benchmarks/*.md` và REFLECTION này từ chính số liệu đã đo. Mọi số liệu do script sinh ra từ máy tôi, không sửa tay; log thô của các lần chạy được giữ trong `benchmarks/`. Các nhận định về cơ chế nào chưa đo trực tiếp đều được ghi rõ là suy luận.
