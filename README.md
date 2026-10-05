# Day 20 — Model Serving & Inference Optimization (Track 2)

Lab cho **AICB-P2T2 · Ngày 20** · Level 3.

> **Hình thức: BÀI CÁ NHÂN.** Mỗi học viên tự làm trên máy của mình và **tự nộp repo
> riêng**, đặt tên `K4-L3-DAY20-HoVaTen-MSSV-ModelServing`.
> **Deadline: 23:59 (giờ Việt Nam, UTC+7) ngày làm lab**, trừ khi key coach thông báo
> khác trong vòng 48 giờ sau buổi lab. Chi tiết: [docs/SUBMISSION.md](docs/SUBMISSION.md).

Bạn dựng một inference stack thật trên laptop của mình, đo **TTFT / TPOT / P50 / P95 /
P99**, đẩy nó tới điểm bão hoà bằng load test, rồi tune một knob và viết report về thay
đổi tạo ra speedup lớn nhất **trên chính máy bạn**.

## Mục tiêu học tập

Sau lab này bạn có thể:

1. Tách latency của LLM thành **TTFT (prefill)** và **TPOT (decode)**, đo P50/P95/P99 và
   giải thích vì sao decode bị chặn bởi **memory bandwidth** chứ không phải FLOPs.
2. Đánh giá đánh đổi **quantization** (4-bit vs 2-bit) theo cả tốc độ lẫn chất lượng.
3. Vận hành một server **OpenAI-compatible** có continuous batching và Prometheus
   `/metrics`, rồi chứng minh batching đang hoạt động bằng số liệu.
4. Load test, xác định **điểm bão hoà**, và dùng **Little's Law** để tách queue time
   khỏi compute time — lập luận goodput@SLO.
5. Tune một knob (thread count, quantization, `--parallel`, …) và giải thích speedup
   bằng **cơ chế**, không phải cảm nhận.
6. Nối serving endpoint vào RAG pipeline và đo latency theo từng stage.

## Thời lượng

| Phần | Thời gian |
|---|---|
| Setup (tải runtime + model) | ~20 phút |
| Base track (100 điểm, bắt buộc) | ~2 giờ |
| Bonus track (tối đa +10 điểm, optional) | ~1–2 giờ |
| Kiểm tra + nộp | ~5 phút |

## Tài liệu

| File | Nội dung |
|---|---|
| 👉 **[docs/GUIDE.md](docs/GUIDE.md)** | **Bắt đầu ở đây** — hướng dẫn từng bước, lệnh cụ thể |
| [docs/CHECKPOINTS.md](docs/CHECKPOINTS.md) | Từng checkpoint: làm gì, sản phẩm gì, cần hiểu gì, tự kiểm tra thế nào |
| [docs/RUBRIC.md](docs/RUBRIC.md) | Tiêu chí chấm, điểm, bằng chứng cần có, lỗi mất điểm — **đọc trước** |
| [docs/SUBMISSION.md](docs/SUBMISSION.md) | Tên repo, file phải nộp, nơi nộp, deadline, kiểm tra trước khi nộp |
| [docs/RULES.md](docs/RULES.md) | Quy định làm bài, dùng AI, sao chép, nộp muộn, bảo mật |
| [docs/HARDWARE-GUIDE.md](docs/HARDWARE-GUIDE.md) | Yêu cầu máy, chọn model, chọn runtime |
| [docs/CLOUD.md](docs/CLOUD.md) | Fallback Colab/Kaggle cho máy < 4 GB RAM |
| [docs/bonus/README.md](docs/bonus/README.md) | Bonus track |

## Cấu trúc repo

```
README.md           ← bạn đang ở đây
docs/               ← toàn bộ tài liệu: GUIDE, RUBRIC, SUBMISSION, RULES, CHECKPOINTS,
                      hướng dẫn từng track (docs/labs/) và bonus (docs/bonus/)
labs/               ← script của từng track: 00-setup · 01-measure · 02-serve · 03-integrate
bonus/              ← script bonus (sweeps, compare-builds, serving regimes, mlx)
lib/labkit.py       ← code dùng chung
scripts/verify.py   ← kiểm tra bài trước khi nộp
cloud/              ← notebook Colab/Kaggle
benchmarks/         ← report do các lệnh sinh ra (bạn commit)
submission/         ← REFLECTION.md + screenshots (bạn điền và commit)
Makefile · lab.ps1  ← lệnh chạy lab (macOS/Linux · Windows)
```

---

## Yêu cầu chuẩn bị — lab này chạy trên máy nào cũng được

| | |
|---|---|
| **Model** | Chọn **một** trong hai (cả hai Apache-2.0, không gated): |
| | **Gemma 4 E2B** — [unsloth/gemma-4-E2B-it-GGUF](https://huggingface.co/unsloth/gemma-4-E2B-it-GGUF) · ~5.2 GB · cần 8 GB RAM · *mặc định* |
| | **Qwen3.5 0.8B** — [unsloth/Qwen3.5-0.8B-GGUF](https://huggingface.co/unsloth/Qwen3.5-0.8B-GGUF) · ~0.9 GB · cần 4 GB RAM · nhanh hơn, nhẹ hơn |
| **Runtime** | **llama.cpp prebuilt binary** — tải 11–33 MB (bản Windows CUDA 140–240 MB), **không compile** |
| **Cần** | Python ≥ 3.10 · **8 GB RAM** (hoặc **4 GB** với Qwen3.5 0.8B) · 3–10 GB đĩa |
| **Không cần** | GPU · compiler · Docker · API key · tài khoản trả phí |
| **OS** | Windows · macOS (Intel + Apple Silicon) · Linux |
| **Windows** | Không có `make` → dùng **`.\lab.ps1 <target>`** (tên target giống hệt) |
| **RAM < 8 GB?** | `LAB_MODEL=qwen35-0.8b make setup` — chạy local với model nhỏ |
| **RAM < 4 GB?** | Dùng [`docs/CLOUD.md`](docs/CLOUD.md) — Colab/Kaggle, **điểm không đổi** |

> **Số liệu của bạn không so sánh được với bạn cùng lớp.** Chỉ so **before vs after trên
> chính máy bạn**. Rubric chấm độ rõ ràng của setup + đo lường + **lập luận**, không chấm
> tốc độ tuyệt đối. Một bạn dùng Air M1 8 GB và một bạn dùng RTX 5090 đều có thể đạt
> 100/100. Toàn bộ 100 điểm base **không cần GPU, không cần compiler**.

---

## Luồng làm lab

Làm **theo đúng thứ tự này**. Đừng nhảy vào bonus trước khi base xong.

```
┌─ 1 ─ BASE TRACK ────────────────── 100 điểm · bắt buộc · ~2 giờ ─┐
│                                                                  │
│   make probe                 hardware.json                       │
│   make setup                 runtime + Gemma 4 E2B               │
│   make bench                 TTFT / TPOT / percentiles, 2 quant  │
│   make tune                  thread sweep → before/after của bạn │
│   make serve   + make smoke  OpenAI-compat API + /metrics        │
│   make load-10 / load-50     load test                           │
│   make metrics               continuous batching (chạy CÙNG load)│
│   make load-report           server bão hoà ở đâu                │
│   make pipeline              RAG → llama-server                  │
│   viết submission/REFLECTION.md                                  │
│   make verify                phải exit 0                         │
└──────────────────────────────────────────────────────────────────┘
                                 │
                    base xong, verify exit 0
                                 ▼
┌─ 2 ─ BONUS TRACK ──────── tối đa +10 điểm · optional · ~1-2 giờ ─┐
│                                                                  │
│   B1  make build-llama && make compare-builds                    │
│       compile cho CPU của bạn → so với prebuilt binary           │
│       (máy yếu thắng đậm nhất ở đây)                             │
│   B2  make sweep-quant / sweep-ctx / sweep-batch / sweep-gpu     │
│   B3  ghi before/after vào REFLECTION §6                         │
│   B4  1 challenge trong docs/bonus/CHALLENGES.md (C1–C7)              │
│   B5  make mlx-compare  ·  make semantic-cache  ·  make embed-demo│
│       (4 lựa chọn — mọi nền tảng đều đạt được)                   │
│                                                                  │
│   Chọn 1-2 cái. MỘT insight giải thích rõ > năm bảng số nông.     │
└──────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─ 3 ─ SUBMIT ──────────────────────────────────────── ~5 phút ────┐
│   make verify → exit 0  ·  repo PUBLIC  ·  paste URL vào LMS      │
│   tên repo: K4-L3-DAY20-HoVaTen-MSSV-ModelServing                │
│   deadline: 23:59 (UTC+7) ngày làm lab — xem docs/SUBMISSION.md       │
└──────────────────────────────────────────────────────────────────┘
```

Chạy `make` để xem toàn bộ target.

---

## Vì sao lab không dùng vLLM / SGLang

Những engine đó cần CUDA GPU + 16 GB VRAM trở lên. Đẹp trên slide, không chạy được trên
một lớp 30 laptop hỗn hợp. llama.cpp cho bạn **cùng teaching surface** — GGUF
quantization, paged KV cache, continuous batching, OpenAI-compat API, Prometheus
`/metrics` — trên bất cứ phần cứng nào bạn đang có.

**Và vì sao prebuilt binary, không phải `llama-cpp-python`:** Gemma 4 dùng architecture
`gemma4` (4/2026). Wheel `llama-cpp-python` trên PyPI vendor một bản llama.cpp cũ hơn và
sẽ báo `unknown model architecture: 'gemma4'`. Prebuilt release binary không có vấn đề
đó, tải nhanh hơn, **và** cho bạn `/metrics` + `--parallel` + `--cont-batching` ngay từ
đầu — những thứ bản Python không có.
