# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Darwin-arm64` · llama.cpp `b10488`
Settings: `threads=10` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 1075 | 56 / 62 | 10.5 / 10.8 | 702 / 739 / 739 | 95.4 |
| UD-Q2_K_XL | 0.39 | 1027 | 54 / 63 | 9.7 / 10.7 | 662 / 732 / 732 | 103.4 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.08x faster** than `Q4_K_M` here, for 0.11 GB less on disk.

## Your observation

**Tốc độ: 2-bit nhanh hơn nhưng rất ít.** Trên Apple M4 (Metal, 10 thread), `UD-Q2_K_XL`
có TPOT P50 9.7 ms so với 10.5 ms của `Q4_K_M` (decode 103.4 so với 95.4 tok/s, nhanh
hơn **1.08×**, tức khoảng 8%). TTFT gần như không đổi (54 so với 56 ms) vì prefill là
compute-bound, không phụ thuộc số byte của weight. File nhỏ hơn 22% (0.39 so với
0.50 GB) nhưng chỉ nhanh hơn 8%, nên decode ở đây **không** tỉ lệ thuận với số byte phải
đọc. Ước tính thô: 0.50 GB × 95.4 tok/s ≈ 48 GB/s, thấp hơn nhiều so với băng thông
~120 GB/s của M4, nên với model 0.8B decode bị chặn một phần bởi chi phí cố định mỗi token
(launch kernel cho từng layer, sampling, HTTP/SSE) và chi phí dequantize của Q2_K — không
chỉ bởi memory bandwidth. Đây là giả thuyết từ phép tính, tôi chưa đo riêng từng thành phần.

**Chất lượng: không thấy khác biệt đủ rõ để biện minh cho 2-bit.** Tôi hỏi cùng bộ câu
hỏi cho cả hai (temperature 0): cả hai trả lời đúng thủ đô Hà Nội, cả hai bịa khi giải
thích "decode bị memory-bandwidth bound" và bịa khi giới thiệu PTIT bằng tiếng Việt. Với
17×23, cả hai ra đúng 391 khi ép "chỉ đưa đáp án cuối" ở temperature 0, còn ở temperature
0.7 (5 lần) mỗi bản chỉ đúng 1/5 lần; khi yêu cầu trình bày từng bước thì bản 2-bit
ra 381 (sai), còn câu trả lời của bản 4-bit bị cắt ở 120 token nên tôi không có kết quả để so. Mẫu quá nhỏ để kết luận 2-bit tệ hơn một cách có ý nghĩa thống kê — nhưng cũng
không có bằng chứng nó tốt hơn.

**Kết luận: không đáng.** Tiết kiệm 0.11 GB và ~8% tốc độ, đổi lại rủi ro chất lượng trên
một model vốn đã nhỏ (0.8B) và nhạy với lượng tử hoá. Với RAM 16 GB, 0.11 GB là không đáng
kể. Bản 4-bit (Q4_K_M) là lựa chọn mặc định hợp lý ở đây. 2-bit chỉ đáng khi RAM thật sự
chật (ví dụ model lớn hơn không vừa bộ nhớ), khi đó lợi ích là "chạy được hay không" chứ
không phải tốc độ.

**Về độ ổn định của số đo:** lần chạy `make bench` đầu tiên của tôi cho `Q4_K_M` load
2076 ms và TTFT P95 = 203 ms (P50 vẫn 56 ms), tức một outlier. Lần chạy thứ hai (số trong
bảng trên) load 1075 ms và TTFT P95 = 62 ms. Khác biệt này cho thấy lần đầu OS page cache
chưa có weights; tôi báo lần chạy thứ hai vì nó phản ánh trạng thái "đã warm". Tôi chưa xác
định chắc chắn nguyên nhân của outlier 203 ms, chỉ thấy nó biến mất khi cache đã ấm.
