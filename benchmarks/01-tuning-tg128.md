# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Darwin-arm64` · llama.cpp `b10488`
CPU: **10 physical · 10 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 104.1 | 100% |
| 5 | 103.0 | 99% |
| 10 | 99.2 | 95% |
| 20 | 81.5 | 78% |

**Best**: `-t 1` at 104.1 tok/s
**Slowest tested**: `-t 20` at 81.5 tok/s (1.28x spread)
**Against the physical-core default** (`-t 10`, 99.2 tok/s): 1.05x

Use this in your run:

```bash
LAB_N_THREADS=1 make bench
```

## Your explanation

**Đường cong của tôi KHÔNG đúng hình dạng kỳ vọng.** Kỳ vọng là throughput tăng tới số
core vật lý rồi đi ngang/giảm. Ở đây (bảng trên, `-ngl 99`) nó phẳng từ 1 đến 5 thread
(104.1 → 103.0 tok/s), giảm nhẹ ở 10 thread (99.2, −5%) và giảm rõ ở 20 thread (81.5,
−22%). Không có "knee" ở số core; peak nằm ở `-t 1`.

**Nguyên nhân: với `-ngl 99` toàn bộ layer chạy trên GPU Metal, không phải trên CPU.**
Tính toán decode (đọc weight, nhân ma trận) do GPU làm; thread CPU chỉ dựng và gửi
command buffer, nên thêm thread không thêm được công việc hữu ích nào. Thread thừa chỉ
thêm chi phí đồng bộ và tranh lịch với các tiến trình khác, vì vậy càng nhiều thread càng
chậm. Đây là lý do `-t 1` đạt tốt nhất. Chênh lệch 1.05× giữa `-t 1` và mặc định `-t 10` nhỏ
so với độ lệch chuẩn của llama-bench (~3–4 tok/s ở r=6), nên riêng một lần đo đó chưa đủ
kết luận.

**Tôi kiểm chứng giả thuyết bằng hai phép đo thêm** (chạy `llama-bench` trực tiếp, không
nằm trong `make tune`):

1. *Lặp lại ngl=99 với r=6.* Hai lượt cho cùng thứ tự: `-t 1` 83.8 / 84.9, `-t 4` 84.7,
   `-t 10` 70.5 / 72.6 tok/s (stddev 3–7), tức 10 thread chậm hơn 1–4 thread khoảng 15%, nhất
   quán qua các lượt. Số tuyệt đối thấp hơn lần `make tune` (≈84 so với ≈104); tôi
   nghi là do máy nóng/throttle sau gần nửa giờ chạy CPU ~800% ở phép đo 2 (chưa đo nhiệt
   độ để xác nhận), nên chỉ dùng thứ tự tương đối trong cùng một lượt, không so số tuyệt đối
   giữa các lượt.
2. *CPU thuần (`-ngl 0`, r=3):* `-t 1` 57.0 · `-t 2` 95.2 · `-t 4` 100.0 · **`-t 6` 104.2** ·
   `-t 8` 94.2 · `-t 10` 57.7 tok/s. Đây mới là đường cong kỳ vọng: tăng mạnh tới 2 thread
   (1.67×), thoải dần tới đỉnh ở 6 thread, rồi **sụp ở 10 thread** về mức của 1 thread. Điểm
   `-t 20` (oversubscribe gấp đôi số core) không hoàn tất: tôi dừng nó sau hơn 23 phút trong
   khi các điểm khác mất vài giây, nên không có số đo cho điểm này.

**Giải thích cho phép đo CPU thuần (có phần là giả thuyết):** (a) từ 1 lên 2–4 thread throughput
tăng nhanh vì một core đơn không kéo đủ băng thông bộ nhớ; (b) sau ~4 thread đường cong gần
như phẳng (≈100 tok/s × 0.5 GB ≈ 50 GB/s), là dấu hiệu chạm trần băng thông mà cụm CPU kéo
được — decode bị chặn bởi memory bandwidth chứ không phải FLOPs; (c) sụp ở 10 thread: M4
10-core gồm 4 performance + 6 efficiency core, và mỗi op của llama.cpp kết thúc bằng một
barrier giữa các thread, nên tốc độ bị quyết định bởi thread chậm nhất (core E, hoặc thread
bị OS preempt). Dùng hết 10 core nghĩa là luôn phải chờ core chậm nhất. Với 20 thread, nhiều
thread hơn core nên các thread spin-wait ở barrier bị preempt, và chạy gần như đứng. Phần
(c) tôi suy ra từ cấu trúc core và hình dạng đường cong, chưa đo trực tiếp (ví dụ chưa
ghim thread vào P-core để xác nhận).

**Kết luận thực tế:** (1) với Metal offload, số thread gần như không quan trọng, chỉ cần
tránh dùng quá nhiều (`-t 1`–`5` tốt hơn `-t 10`/`20`); (2) với CPU thuần, dùng ~4–6 thread
chứ không phải cả 10 core; (3) đáng chú ý là ở decode (`tg128`) với model 0.8B này, CPU
thuần ở 6 thread (104.2 tok/s) **ngang** với Metal (104.1 tok/s, lượt `make tune`), nên tôi
không có bằng chứng rằng offload GPU làm decode nhanh hơn. Lợi thế của GPU, nếu có, nhiều
khả năng nằm ở prefill (compute-bound), chưa được đo ở đây (`tune.py --metric pp512`).
