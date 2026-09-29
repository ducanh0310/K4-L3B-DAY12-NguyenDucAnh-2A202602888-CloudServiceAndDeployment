# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Đức Anh  Mã học viên: 2A202602888

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống: Khi deploy service lên môi trường cloud (Railway, Render hoặc K8s), người vận hành quên cấu hình biến môi trường `AGENT_API_KEY` trong dashboard. 
- Nếu có giá trị mặc định là `"changeme"`: Service vẫn khởi động bình thường. Kẻ xấu có thể thử các từ khóa mặc định phổ biến như `"changeme"` để gọi API trái phép, làm rò rỉ dữ liệu hoặc bào mòn tài nguyên/chi phí token LLM. Ta chỉ phát hiện ra khi đã mất tiền hoặc lộ lọt thông tin.
- Khi không có giá trị mặc định (Fail Fast): App lập tức văng `ValidationError` và crash ngay lúc khởi động (container báo unhealthy). Hệ thống deployment lập tức báo lỗi đỏ, bắt buộc dev/ops phải cung cấp secret hợp lệ trước khi cho phép đón nhận traffic công khai, ngăn chặn hoàn toàn rủi ro bảo mật từ đầu.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Log JSON mẫu:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T03:07:11.123456+00:00", "user_id": "user_123", "tokens_in": 15, "tokens_out": 42, "cost_usd": 0.00057}`

Hai việc làm được với dòng log JSON có cấu trúc mà `print("đã trả lời xong")` không làm được:
1. **Truy vấn, tổng hợp và cảnh báo tự động bằng công cụ quản lý log (Datadog, Loki, CloudWatch, Elasticsearch):** Do log có định dạng JSON gồm các trường có kiểu dữ liệu rõ ràng (`cost_usd`, `tokens_in`, `tokens_out`, `user_id`), hệ thống có thể tự động parse dữ liệu để vẽ biểu đồ chi phí thời gian thực, tính tổng ngân sách tiêu thụ theo từng user, hoặc kích hoạt alert khi một request tốn vượt mức chi phí mà không cần dùng regex phân tích văn bản thô.
2. **Lọc và truy vết theo ngữ cảnh (Structured Filtering & Auditing):** Nhờ có `level: "info"`, `timestamp` chuẩn ISO-8601 UTC và `user_id`, ta có thể dễ dàng lọc log theo khung thời gian chính xác, lọc riêng biệt mức độ log (DEBUG/INFO/ERROR) và trace toàn bộ chuỗi hành vi của một user cụ thể khi cần điều tra sự cố. Trong khi đó, `print("đã trả lời xong")` là văn bản phi cấu trúc, thiếu timestamp, thiếu context và không thể phân loại theo level.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | ... MB |
| Multi-stage | ... MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> *Câu trả lời của bạn*

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> *Câu trả lời của bạn*

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> *Câu trả lời của bạn*

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> *Câu trả lời của bạn*

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> *Câu trả lời của bạn*

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> *Câu trả lời của bạn*

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> *Câu trả lời của bạn*

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> *Câu trả lời của bạn*
