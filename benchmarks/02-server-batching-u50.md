# 02 - Continuous batching under load (u50)

Host `Darwin-arm64` · `--parallel 4` · 30 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.96 of 4 slots (99%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 12707 |

Highest sampled value was **3.96 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

**Peak batch width = 3.96 / 4 slot (99%)**, và continuous batching đang hoạt động: gauge luôn
ở 3.92–3.96, `requests_processing` = 4 trong hầu hết các mẫu. Từ chính CSV: giữa mẫu đầu và mẫu
cuối (58.6 s), `n_decode_total` tăng 1674 → 3280 (1606 decode step) và `tokens_predicted_total`
tăng 6506 → 12707 (6201 token), tức **~3.86 token mỗi decode step**, đúng bằng việc 4 request
chia sẻ chung mỗi bước decode.

**Có khớp với effective concurrency (38.2) trong `02-server-results.md` không?** Hai con số
**không mâu thuẫn**, chúng đo hai thứ khác nhau. 3.96 là số request *đang được phục vụ* (không
thể vượt 4 vì chỉ có 4 slot), còn 38.2 (Little's Law) đếm cả request *xếp hàng*. Phần chênh
~34 chính là hàng đợi, và nó khớp với `requests_deferred` ≈ 44 (4 đang chạy + ~44 chờ ≈ 48
≈ 50 users). Tôi tin gauge của server hơn khi hỏi "slot có bận không", và tin Little's Law
hơn khi hỏi "có bao nhiêu request đang ở trong hệ thống".

**Hai lưu ý để không đọc quá mức:** (1) `n_busy_slots_per_decode` là *trung bình tích luỹ từ lúc
server khởi động* chứ không phải giá trị tức thời; mẫu đầu đã là 3.92 vì server đã chịu lần
`load-10` và smoke test trước đó, nên con số này không tách riêng được cửa sổ 50 users. (2) Batch
width cao **không** đồng nghĩa với throughput cao: ở thí nghiệm `02-parallel-experiment.md`, tổng
throughput chỉ ~105 tok/s ở `--parallel 4` và ~111 tok/s ở `--parallel 8`, xấp xỉ tốc độ
decode của một luồng đơn (~95–104 tok/s), dù mỗi step gộp 4 hay ~8 token.
