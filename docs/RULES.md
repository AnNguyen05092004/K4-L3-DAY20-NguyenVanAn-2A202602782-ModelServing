# Quy định làm bài — Day 20 Lab

## 1. Hình thức

- **Bài cá nhân.** Mỗi học viên làm trên máy của mình (hoặc Colab/Kaggle nếu máy dưới
  4 GB RAM) và **tự nộp một repo riêng**, đặt tên theo [docs/SUBMISSION.md](SUBMISSION.md).
- Repo nộp phải **public** cho đến khi điểm được công bố.

## 2. Số liệu phải là của bạn

- Mọi con số trong `benchmarks/*.md` và `REFLECTION.md` phải sinh ra từ **máy bạn đã khai
  báo** trong `hardware.json` và REFLECTION §1.
- Không sửa tay số liệu do script sinh ra. Nếu một lần chạy hỏng, chạy lại và nói rõ
  trong REFLECTION.
- Kết quả trái kỳ vọng **không** bị trừ điểm — được thưởng điểm nếu giải thích đúng. Không
  có lý do gì để "làm đẹp" số.

## 3. Sử dụng AI

- **Được phép** dùng công cụ AI (ChatGPT, Claude, Copilot, …) để hiểu khái niệm, đọc lỗi,
  debug lệnh và sửa code stub trong `pipeline.py`.
- **Không được** dùng AI để bịa số liệu, viết hộ toàn bộ phần lập luận mà bạn không hiểu,
  hoặc tạo screenshot giả.
- Phần giải thích cơ chế (REFLECTION §5, các section "required" trong `benchmarks/*.md`)
  là **lập luận của bạn về số liệu của bạn**. Grader/coach có thể hỏi lại trực tiếp; nếu
  bạn không giải thích được điều mình đã viết, phần đó có thể bị chấm 0.
- **Khai báo** ngắn ở REFLECTION §9: công cụ nào, dùng vào việc gì.

## 4. Hợp tác và sao chép

- Được trao đổi ý tưởng, cách cài đặt, cách đọc lỗi với bạn cùng lớp.
- **Không được** sao chép số liệu, screenshot, file `benchmarks/*.md` hay văn bản
  REFLECTION của người khác. Số liệu gắn với phần cứng nên sao chép rất dễ bị phát hiện
  (lệch với `hardware.json`).
- Bài sao chép: **0 điểm** cho phần bị sao chép ở **tất cả** các bài liên quan, và có thể
  bị xử lý theo quy chế học vụ.

## 5. Deadline và nộp muộn

- **Deadline mặc định: 23:59 (giờ Việt Nam, UTC+7) ngày làm lab.**
- Nếu key coach thông báo deadline khác trong vòng **48 giờ** sau buổi lab thì áp dụng
  thông báo đó.
- Nộp sau deadline là **nộp muộn và bị trừ điểm** theo mức key coach công bố.
- Bài chỉ được tính là đã nộp khi **URL repo đã được paste vào LMS** trước deadline.

## 6. Sửa bài sau deadline

- Grader chấm **commit cuối cùng trước deadline**.
- Commit push sau deadline **không** được tính, trừ khi bạn chủ động báo coach để chấm
  như một bài nộp muộn (áp dụng mức trừ điểm ở mục 5).
- Không force-push / viết lại lịch sử git sau deadline. Bài có lịch sử bị viết lại sau
  deadline được xem là nộp muộn.

## 7. Bảo mật API key và dữ liệu

- Lab **không cần** API key nào. `HF_TOKEN` chỉ dùng khi bị Hugging Face rate-limit.
- **Không bao giờ** commit `.env`, `HF_TOKEN`, token GitHub hay bất kỳ secret nào.
  `.env` đã có trong `.gitignore`; set biến môi trường inline thay vì ghi vào file
  được track.
- Nếu lỡ commit secret: **revoke/rotate key ngay** — xoá khỏi commit sau là không đủ vì
  lịch sử git đã public.
- Nếu nối dữ liệu N16–N19 thật vào `pipeline.py`: **không** commit dữ liệu cá nhân, dữ
  liệu nội bộ hay dữ liệu có bản quyền. Dùng dữ liệu mẫu hoặc mô tả trong REFLECTION.
- Không commit model weights (`models/*.gguf`) hay binary (`runtime/`) — chúng đã được
  `.gitignore` và grader không cần.

## 8. Bonus

Bonus là điểm cộng **cho bài lab** (tối đa **10/100**), chấm từ bằng chứng trong repo
theo [docs/RUBRIC.md](RUBRIC.md). Không phải điểm giơ tay, phát biểu hay pitching.
