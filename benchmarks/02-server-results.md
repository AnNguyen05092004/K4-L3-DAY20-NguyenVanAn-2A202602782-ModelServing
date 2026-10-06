# 02 - Serve: load test + saturation reading

Host `Darwin-arm64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=10` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 98 | 1.87 | 3600 | 5900 | 6500 | 7.0 | 0.0% |
| 50 | 114 | 1.93 | 24000 | 27000 | 28000 | 38.2 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.03x** (21% of linear) |
| P95 latency | **4.58x** |
| Effective concurrency at 50 users | 38.2 vs `--parallel 4` slots (occupancy/slot ratio 9.55) |

**Saturated.** Throughput delivered only 1.03x for 5x the offered load, and effective concurrency (38.2) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.03x while P95 moved 4.58x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

**Server bão hoà ở đâu: sớm hơn nhiều so với 50 users, nhiều khả năng ngay quanh số slot
(4).** Con số thuyết phục nhất là **effective concurrency ở chỉ 10 users đã là 7.0**, lớn hơn
4 slot (script tự sinh ra nói "saturation ở ≤ 50 users", nhưng số liệu của tôi cho thấy nó đã
xảy ra ở ≤ 10 users). Bằng chứng đi kèm: RPS gần như đứng yên khi tăng tải: 1.87 (10 users)
→ 1.93 (50 users), tức ×1.03 cho tải ×5; trong khi P95 tăng ×4.58 (5.9 s → 27 s) và P50
tăng từ 3.6 s lên 24 s. Tôi chỉ đo hai điểm (10 và 50) nên **không xác định được chính xác
knee**; chỉ biết nó nằm ở ≤ 10 users.

**Phần latency tăng thêm là queue time, không phải compute.** Một request đơn lẻ chỉ tốn
~0.5–1 s (48–96 token ở ~10 ms/token, `01-quickstart-results.md`), nhưng ở 50 users trung bình
mất 19.8 s. Khoảng chênh ~19 s không thể là compute. Nó khớp với phép đếm hàng đợi của server
(`02-server-batching-u50.md`): lúc đó `requests_processing` = 4 và `requests_deferred` ≈ 44,
cộng lại ≈ 48 request trong hệ thống, gần bằng 50 users. Tức là 4 request đang chạy, ~44 xếp
hàng chờ slot.

**Goodput@SLO.** Tôi chọn SLO giả định là P95 ≤ 10 s (tài liệu không cố định con số). Ở 10
users P95 = 5.9 s nên đạt SLO, goodput ≈ throughput ≈ 1.87 RPS. Ở 50 users P50 đã là 24 s nên
**hơn một nửa số request vượt SLO**; throughput 1.93 RPS vẫn "đẹp" nhưng phần lớn request đó
không còn được phục vụ trong SLO, nên goodput@SLO thấp hơn nhiều so với 1.93 (tôi không có
phân phối dưới P50 để tính chính xác). Đây chính là khác biệt giữa peak throughput và goodput.

**Knob đổi trước: không phải `--parallel`, mà là số token sinh ra mỗi request
(`max_tokens`/độ dài câu trả lời).** Lý do dựa trên thí nghiệm bổ sung ở
`02-parallel-experiment.md`: tăng `--parallel` từ 4 lên 8 (với `ctx-size` tăng theo để
không gây lỗi 400) chỉ tăng RPS ×1.06 (1.85 → 1.97), vì tổng throughput đã chạm trần ~105–111
tok/s của máy này. Khi tok/s tổng là hằng số, RPS = tok/s ÷ token mỗi request, nên cách duy
nhất để tăng RPS (và giảm hàng đợi) là giảm số token mỗi request. Hiện trung bình ~58 token
mỗi request (6290 token / 108 request ở lần chạy `--parallel 4`). Đây là suy luận từ số liệu,
**tôi chưa chạy thí nghiệm hạ `max_tokens`** để xác nhận. Cùng lúc, vì mọi thứ quá ngưỡng ~4–7
request đồng thời chỉ biến thành hàng đợi, chặn bớt request ở lớp ngoài (admission control /
load shedding) sẽ giữ P95 trong SLO; llama-server không có sẵn knob này nên phải làm ở gateway.
