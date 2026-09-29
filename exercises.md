# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng mẫu bên dưới mỗi câu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Ngô Minh Thu. Mã học viên: 2A202602679.

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Ví dụ deploy bị quên `AGENT_API_KEY`: fail fast làm service dừng ngay và log
chỉ rõ lỗi cấu hình. Nếu dùng `changeme`, service vẫn chạy nhưng ai đoán được
key mặc định có thể gọi API và phát sinh chi phí.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

`{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T10:01:11.793014+00:00", "user_id": "sv01", "cost_usd": 0.0001}`
Có thể lọc/đếm request theo `user_id` và tạo cảnh báo theo `cost_usd` hoặc
`level`; dòng `print` tự do không cho máy đọc các trường này một cách ổn định.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản               | Dung lượng                         |
| ----------------- | ---------------------------------- |
| 1 stage (bản đầu) | Chưa đo — Docker Desktop chưa chạy |
| Multi-stage       | Chưa đo — Docker Desktop chưa chạy |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Multi-stage chỉ đưa dependency đã cài sang runtime; không mang compiler,
cache pip và build context thừa. Cần chạy lại hai lệnh build khi Docker Desktop
hoạt động để điền số MB thật.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Khi chỉ sửa `app/main.py`, các layer `FROM`, `COPY requirements.txt` và `RUN
pip install` dùng cache; `COPY app` và các layer sau chạy lại. Nếu `COPY . .`
đứng trước `pip install`, mọi sửa code làm mất cache và cài dependency lại.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Lỗ hổng có thể cho kẻ tấn công thực thi lệnh trong container. Nếu tiến trình là
root, họ có đặc quyền root trong container và rủi ro leo thang khi host cấu hình
sai. `USER appuser` hạ quyền tiến trình, nên lệnh khai thác không có quyền root.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Tối đa 20 request: gửi 10 request ở giây 59 của một phút và 10 request ở giây
00 của phút kế tiếp. Bộ đếm theo phút đã reset, dù chỉ cách nhau khoảng 2 giây.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn số request trong 60 giây; cost guard giới hạn tiền theo
tháng. Request ít nhưng rất tốn token có thể qua rate limit và bị cost guard
chặn. Ngược lại, request rẻ nhưng gửi quá nhanh bị rate limit chặn dù còn tiền.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Khi Redis mất 30 giây, endpoint gộp trả lỗi. Orchestrator hiểu nhầm container
chết, lần lượt restart cả 3 container; trong khi process vẫn khỏe và chỉ Redis
đang lỗi. Tách `/health` tránh restart, còn `/ready` chỉ ngừng nhận traffic.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Với Redis, `history_length` tăng đều giữa các request, thường là `0`, rồi `2`,
rồi `4`, dù request vào container nào. Dict Python nằm riêng từng container nên
kết quả có thể quay lại `0` hoặc tăng không đều khi load balancer đổi instance.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Em chưa có lỗi gì ạ
