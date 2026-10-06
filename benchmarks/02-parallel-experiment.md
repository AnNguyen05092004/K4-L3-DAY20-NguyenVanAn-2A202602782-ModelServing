# 02 - Experiment: `--parallel` 1 vs 4 vs 8 at 50 users

Thí nghiệm bổ sung (không nằm trong `make`), theo gợi ý trong `docs/labs/02-serve.md`:
chạy `load-50` ở `--parallel 1`, rồi 4, rồi 8. Host `Darwin-arm64` (Apple M4) · llama.cpp `b10488` ·
Qwen3.5 0.8B `Q4_K_M` · `threads=10` · `ngl=99` · 50 users, ramp 25/s, 60 s mỗi lần.
Mỗi cấu hình khởi động server mới. Raw locust: `02-parallel-p{1,4,8}_stats.csv`.

| `--parallel` | `ctx-size` (mỗi slot) | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Tokens sinh ra (~60 s) | ≈ tok/s tổng | Token / decode step | Failures |
|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|
| 1 | 2048 (2048) | 71 | 1.23 | 27000 | 40000 | 42000 | 3943 | 66 | 0.98 | 0 |
| 4 | 2048 (512) | 108 | 1.85 | 25000 | 28000 | 28000 | 6290 | 105 | 3.92 | 0 |
| 8 | 4096 (512) | 116 | 1.97 | 22000 | 26000 | 28000 | 6651 | 111 | 7.72 | 0 |

`Tokens` = chênh `llamacpp:tokens_predicted_total` trước/sau lần chạy; `Token / decode step`
= chênh `tokens_predicted_total` / chênh `n_decode_total`. Hai cột này đo từ `/metrics` của
server, không phải từ locust. Số `--parallel 4` ở đây là một lần chạy riêng, **không phải**
lần `make load-50` dùng cho `02-server-results.md` (lần đó: 114 req, 1.93 RPS, P95 27000 ms).
Hai lần cho kết quả cùng cỡ (108 so với 114 request), cho thấy độ nhiễu giữa các lần chạy
khoảng 5%.

## Một lần chạy bị loại và vì sao

Lần chạy `--parallel 8` đầu tiên giữ `ctx-size 2048`, nên mỗi slot chỉ còn **256 token**
(`--parallel` chia đều `ctx-size`). Request `long-rag` (286 token) bị server từ chối:
`request (286 tokens) exceeds the available context size (256 tokens)` → HTTP 400, 47
failures, và RPS hiện ra 3.20 một cách giả tạo vì lỗi trả về ngay lập tức. Tôi bỏ lần đó và
chạy lại với `LAB_N_CTX=4096` (mỗi slot 512 token, bằng `--parallel 4`), 0 failures. Bài học:
tăng `--parallel` mà không tăng `ctx-size` thì RPS đẹp hơn có thể chỉ là request lỗi.

## Đọc kết quả

- **1 → 4 slot: có lợi thật.** RPS ×1.5 (1.23 → 1.85), token/s tổng ×1.6 (66 → 105), P95 giảm
  40000 → 28000 ms. Mỗi decode step xử lý ~3.9 token thay vì ~1: đây là continuous batching.
- **4 → 8 slot: gần như không lợi.** RPS chỉ ×1.06 (1.85 → 1.97) và token/s ×1.06 dù mỗi step
  giờ gộp ~7.7 token. Throughput đã chạm trần ở ~105–111 tok/s, **xấp xỉ tốc độ decode của một
  luồng đơn** (~95–104 tok/s, từ `make bench` và `make tune`). Nghĩa là mỗi decode step
  gộp N token tốn thời gian gần tỉ lệ với N: với model 0.8B trên GPU Metal của M4, bước
  decode có vẻ đã bị chặn bởi tính toán/chi phí theo token chứ không còn chỉ bởi việc đọc
  weight, nên gộp thêm request không "miễn phí". (Đây là suy luận từ số liệu trên; tôi chưa
  profile GPU để xác nhận.)
- Vì vậy thêm slot chỉ chuyển thời gian chờ từ hàng đợi vào trong slot; tổng công việc/giây
  gần như không đổi.
