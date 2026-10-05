# Submission — Day 20 Lab

> **Bài cá nhân.** Mỗi học viên tự nộp **một repo riêng**. Không nộp chung, không nộp
> qua repo của người khác.

## 1. Tên repo bài nộp

```
K4-L3-DAY20-HoVaTen-MSSV-ModelServing
```

| Phần | Quy tắc | Ví dụ |
|---|---|---|
| `K4-L3` | Cố định cho lab này | `K4-L3` |
| `DAY20` | Ngày học, hai chữ số | `DAY20` |
| `HoVaTen` | Họ và tên **không dấu, không khoảng trắng**, viết hoa chữ cái đầu mỗi từ | `NguyenVanAn` |
| `MSSV` | Mã số sinh viên của bạn | `20241234` |
| `ModelServing` | Tên bài, cố định | `ModelServing` |

Ví dụ đầy đủ: **`K4-L3-DAY20-NguyenVanAn-20241234-ModelServing`**

Các phần ngăn cách bằng dấu `-`. Repo sai tên dễ bị sót hoặc chấm nhầm.

## 2. Cách tạo repo

1. Tạo repo mới trên GitHub account của bạn với **đúng tên ở mục 1**, chế độ **Public**.
   (Hoặc fork repo đề bài rồi vào *Settings → Repository name* để đổi tên.)
2. Trỏ clone local của bạn tới repo đó:

   ```bash
   git remote set-url origin https://github.com/<you>/K4-L3-DAY20-<HoVaTen>-<MSSV>-ModelServing.git
   git push -u origin main
   ```

Repo phải **public cho đến khi điểm được công bố**. Private → grader không đọc được →
**0 điểm**.

## 3. Cấu trúc repo và các file phải nộp

Grader chỉ thấy **file đã commit**. Tất cả các file dưới đây phải có trong repo:

```
K4-L3-DAY20-HoVaTen-MSSV-ModelServing/
├── hardware.json                              make probe            (rubric 1)
├── models/
│   └── active.json                            make setup            (rubric 2)
├── benchmarks/
│   ├── 01-quickstart-results.md               make bench            (rubric 3, 4, 5)
│   ├── 01-tuning-tg128.md                     make tune             (rubric 11)
│   ├── locust-10_stats.csv                    make load-10          (rubric 8)
│   ├── locust-50_stats.csv                    make load-50          (rubric 8)
│   ├── 02-server-batching-u50.md  + .csv      make metrics          (rubric 9)
│   ├── 02-server-results.md                   make load-report      (rubric 10)
│   ├── 03-integration-results.md              make pipeline         (rubric 12, 13)
│   └── bonus-*.md                             (chỉ khi làm bonus)
└── submission/
    ├── REFLECTION.md                          bạn viết              (rubric 11, 13, 14)
    └── screenshots/                           ≥ 5 ảnh               (rubric 6, 7, 8, 14)
        ├── 01-hardware-probe.png
        ├── 02-bench.png
        ├── 03-serve-and-smoke.png
        ├── 04-locust-10.png
        └── 05-locust-50.png
```

Mỗi file `benchmarks/*.md` có một section **"required -- replace this line"** — bạn
phải thay bằng nhận xét của mình. Chi tiết screenshot:
[`submission/screenshots/README.md`](../submission/screenshots/README.md).

**Không** commit: `models/*.gguf`, `runtime/`, `.venv/`, `.env`. Các path này đã có trong
`.gitignore`; đừng `git add -f`.

## 4. Nơi nộp

Paste **URL public của repo** vào ô submission **Day 20** trên **VinUni LMS**. Không cần
mở Pull Request.

## 5. Deadline

**23:59 (giờ Việt Nam, UTC+7) ngày làm lab.**

Nếu key coach thông báo deadline khác trong vòng **48 giờ** sau buổi lab thì theo thông
báo đó. Sau deadline là **nộp muộn và bị trừ điểm** — xem [docs/RULES.md](RULES.md).

Grader chấm **commit cuối cùng trước deadline**.

## 6. Kiểm tra trước khi nộp

```bash
make verify            # Windows: .\lab.ps1 verify
```

Lệnh này phải **exit 0**. Nó kiểm tra: đủ file ở mục 3, file đã được commit, không còn
section "required -- replace this line", REFLECTION §1–5 không còn placeholder hay ô
bảng trống, đủ 5 screenshot.

Sau đó tự kiểm tra thêm những gì `verify` không kiểm được:

- [ ] Tên repo đúng mẫu ở mục 1
- [ ] Repo ở chế độ **Public** (mở URL trong cửa sổ ẩn danh để chắc chắn)
- [ ] Số trong `REFLECTION.md` khớp với `benchmarks/*.md`
- [ ] Đã khai báo Colab/Kaggle trong REFLECTION §1 nếu dùng cloud fallback
- [ ] Đã khai báo real/stub cho N16–N19 trong REFLECTION §4
- [ ] Đã push commit cuối cùng **trước** 23:59 (UTC+7)
- [ ] Đã paste URL vào LMS
