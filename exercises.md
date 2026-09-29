# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `*Câu trả lời của bạn*` bằng câu trả lời.
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
| 1 stage (bản đầu) | ~1020 MB (1.02 GB) |
| Multi-stage | ~185 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~835 MB) bao gồm:
1. **Base Image tối giản**: Bản 1 stage dùng `python:3.11` đầy đủ dựa trên Debian chuẩn, tích hợp sẵn toàn bộ công cụ build/biên dịch (gcc, g++, make), thư viện C header phát triển và rất nhiều công cụ hệ thống không cần thiết cho môi trường chạy production. Bản multi-stage dùng `python:3.11-slim`, loại bỏ hoàn toàn các compiler và gói công cụ dư thừa này.
2. **Loại bỏ Build Artifacts & Pip Cache**: Ở bản 1 stage, toàn bộ file nén tải về, cache của pip và các file tạm sinh ra trong lúc compile nằm lại vĩnh viễn trong các layer image. Ngược lại, mô hình multi-stage cô lập quá trình build ở stage `builder`, stage `runtime` chỉ sao chép thư mục sản phẩm sạch (`/install` sang `/usr/local`), loại bỏ toàn bộ cache và artifact trung gian.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile tối ưu hiện tại:
  - Các layer `FROM`, `WORKDIR`, `COPY requirements.txt .` và `RUN pip install ...` hoàn toàn **được dùng lại từ cache** (CACHED) vì checksum của `requirements.txt` không hề thay đổi.
  - Chỉ từ layer `COPY . .` (sao chép source code có `main.py` thay đổi) trở đi mới bị vô hiệu hóa cache (cache invalidated) và phải chạy lại, giúp build cực nhanh chỉ mất 1-2 giây.
- Nếu đặt `COPY . .` lên trước `RUN pip install`:
  - Mỗi khi sửa dù chỉ 1 ký tự trong `main.py`, checksum của thư mục thay đổi làm cho layer `COPY . .` bị cache bust.
  - Theo nguyên lý của Docker, mọi layer đứng sau layer bị thay đổi đều phải thực thi lại. Do đó Docker **buộc phải chạy lại toàn bộ lệnh `RUN pip install` từ đầu**, khiến thời gian build kéo dài nhiều phút và tiêu tốn băng thông vô ích.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện tấn công:
  1. Ứng dụng Python gặp lỗ hổng bảo mật (ví dụ: Command Injection, RCE hoặc lỗi bảo mật trong thư viện bên thứ ba).
  2. Kẻ tấn công kích hoạt lỗ hổng để thực thi mã tùy ý. Nếu container chạy bằng `root` (UID 0), kẻ tấn công ngay lập tức sở hữu toàn quyền root bên trong container (đọc/ghi mọi file hệ thống, can thiệp tiến trình, cài thêm công cụ tấn công).
  3. Từ quyền root trong container, nếu container có mount các volume từ máy host (đặc biệt là Docker socket `/var/run/docker.sock` hoặc thư mục nhạy cảm của host), hoặc nếu nhân Linux Kernel xuất hiện lỗ hổng container breakout (như Dirty COW, cgroup breakout), kẻ tấn công sẽ thoát ra khỏi container (escape) sang máy host với đúng UID 0 (root máy host), qua đó chiếm quyền kiểm soát toàn bộ server.
- Lệnh `USER appuser` cắt đứt chuỗi ở đâu:
  Lệnh `USER appuser` (UID 10001 không đặc quyền) cắt đứt chuỗi ngay tại **bước 2**: Kẻ tấn công dù khai thác được code Python thì shell/tiến trình sinh ra cũng chỉ có quyền của user thường. User này bị cấm chỉnh sửa file hệ thống container, không có Linux capabilities đặc quyền (như `CAP_SYS_ADMIN`), bị chặn không thể tương tác trực tiếp với Docker socket của host hoặc thực thi các kỹ thuật container breakout yêu cầu quyền root.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- Con số tối đa: **20 request** trong 2 giây liên tiếp.
- Giải thích cách đạt được:
  Với cơ chế đếm theo phút đồng hồ (fixed window reset tại giây 00):
  1. Người dùng gửi 10 request dồn vào giây `10:00:59` (giây cuối cùng của phút thứ 10). Lúc này, bộ đếm của phút 10 là 10 request, hoàn toàn hợp lệ (chưa vượt hạn mức 10/phút).
  2. Ngay 1 giây sau đó, đồng hồ chuyển sang `10:01:00` (bắt đầu phút thứ 11). Bộ đếm fixed window tự động reset về 0. Người dùng lập tức gửi tiếp 10 request nữa trong giây `10:01:00` (hoặc `10:01:01`). Lúc này bộ đếm của phút 11 ghi nhận 10 request, vẫn hoàn toàn hợp lệ theo luật.
  3. Kết quả là trong khoảng thời gian chỉ 2 giây liên tiếp (từ `10:00:59` đến `10:01:00`), hệ thống đã phải gánh chịu tới 20 request (gấp đôi hạn mức quy định). Đây là lỗ hổng "burst traffic tại biên" của fixed window. Thuật toán Sliding Window 60 giây (dùng Redis ZSET) giải quyết triệt để lỗi này bằng cách luôn tính chính xác tổng số request trong đúng 60 giây gần nhất tính từ thời điểm gọi.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Điểm khác nhau cốt lõi:
  - **Rate Limit**: Giới hạn **tần suất / số lượng request trong một khoảng thời gian ngắn** (ví dụ: tối đa 10 request / 60 giây) nhằm bảo vệ hạ tầng máy chủ khỏi tình trạng quá tải CPU/RAM, nghẽn mạng hoặc tấn công từ chối dịch vụ (DoS/DDoS). Cơ chế này không quan tâm kích thước nội dung hay chi phí của request.
  - **Cost Guard**: Giới hạn **tổng chi phí tài chính tiêu thụ trong một chu kỳ dài** (ví dụ: tối đa 10.0 USD / tháng) nhằm bảo vệ ví tiền của bạn khỏi hóa đơn API LLM tăng vọt. Cơ chế này tính toán theo lượng token tiêu thụ và số tiền phát sinh, không quan tâm request gửi nhanh hay chậm.
- Tình huống Rate Limit cho qua nhưng Cost Guard chặn:
  - Một người dùng cả tháng mới gửi 1 request duy nhất (tần suất cực thấp: 1 request/phút -> Rate Limit hoàn toàn cho qua). Tuy nhiên, người này trước đó đã tiêu hết 10.0 USD ngân sách của tháng, hoặc request hiện tại có câu hỏi quá dài khiến ước tính chi phí vượt quá ngân sách còn lại. Khi đó Cost Guard sẽ chặn ngay lập tức và trả về mã lỗi `402 Payment Required`.
- Tình huống Cost Guard cho qua nhưng Rate Limit chặn:
  - Một người dùng mới toanh vào đầu tháng, ngân sách còn nguyên 10.0 USD (chưa tiêu đồng nào). Người dùng này chạy script gửi tới tấp 15 câu hỏi ngắn chỉ trong vòng 3 giây. Về mặt chi phí, 15 câu hỏi này chỉ tốn vài cent (rất nhỏ so với 10 USD), nhưng vì gửi quá nhanh vượt quá 10 req/phút, Rate Limit sẽ chặn từ request thứ 11 trở đi và trả về mã lỗi `429 Too Many Requests` để bảo vệ server.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện xảy ra:
1. **Redis mất kết nối trong 30 giây** (do sự cố mạng, restart hoặc quá tải).
2. **Liveness check thất bại**: Orchestrator (Docker/Kubernetes) gửi request định kỳ (mỗi 5-10s) tới endpoint chung. Vì endpoint này kiểm tra Redis và Redis không phản hồi, nó trả về mã lỗi 503 hoặc timeout.
3. **Orchestrator restart toàn bộ container**: Sau vài lần thử lại thất bại liên tiếp (`retries`), orchestrator lầm tưởng rằng tiến trình bên trong cả 3 container agent đều "đã chết" (unhealthy) và ra lệnh **kill rồi restart lại toàn bộ 3 container**.
4. **Xảy ra Restart Storm (CrashLoopBackOff)**: Trong 30 giây Redis chưa hồi phục, cả 3 container sau khi vừa khởi động lại tiếp tục bị liveness probe đánh rớt ➔ lại bị restart liên tục theo vòng lặp. Mọi request của người dùng đang được xử lý dở dang đều bị ngắt quãng giữa chừng (người dùng gặp lỗi 502 Bad Gateway), đồng thời CPU/RAM của server bị tiêu tốn lãng phí vào việc liên tục khởi động lại ứng dụng.
5. **Ý nghĩa của việc tách rời `/health` và `/ready`**: Khi tách riêng, nếu Redis mất kết nối, `/ready` sẽ trả về 503 để Load Balancer tạm thời ngừng điều hướng traffic vào container (chờ Redis hồi phục), nhưng `/health` vẫn trả về 200 để báo rằng bản thân container agent vẫn đang sống khỏe mạnh, tuyệt đối không bị orchestrator restart oan.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Khi lưu trong Redis (Stateless service):
  Mọi instance agent đều đọc và ghi dữ liệu lịch sử vào một database Redis dùng chung bên ngoài process. Do đó, `history_length` sẽ tăng đều đặn, liên tục và nhất quán sau mỗi lượt hỏi: 0 ➔ 2 ➔ 4 ➔ 6 ➔ 8... bất kể request rơi vào instance nào trong 3 container.
- Nếu lưu trong một dict Python (RAM của từng instance):
  Vì Load Balancer phân phối các request luân phiên (Round-Robin) tới 3 container độc lập (Container 1, 2, 3), mỗi container chỉ có một vùng nhớ RAM riêng biệt:
  - Request 1 đến Container 1: `history_length = 0` (Container 1 lưu câu hỏi 1 vào RAM của nó).
  - Request 2 đến Container 2: `history_length = 0` (vì RAM của Container 2 hoàn toàn trống rỗng, chưa từng trò chuyện với user này).
  - Request 3 đến Container 3: `history_length = 0` (Container 3 cũng không hề biết gì về các câu hỏi trước).
  - Request 4 lại rơi vào Container 1: `history_length = 2` (Container 1 nhớ câu hỏi 1).
  ➔ Con số `history_length` sẽ nhảy lộn xộn, agent bị "mất trí nhớ" và không thể duy trì được mạch hội thoại thống nhất với người dùng. Cấu trúc Stateless tách toàn bộ state ra Redis là điều kiện bắt buộc để có thể scale ngang (horizontal scaling).

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Tình huống lỗi:** Lỗi `401 Unauthorized` hoặc `Connection Timeout / Bad Gateway` khi deploy lên Render/Railway.
- **Thông báo lỗi gặp phải:**
  - Trên trình duyệt/curl: `{"detail":"invalid or missing API key"}` (khi kiểm tra probe có xác thực) hoặc `502 Bad Gateway`.
  - Trên logs của Render/Cloud: `Failed to bind to port 10000: port already in use` hoặc `Timed out waiting for health check on port 10000`.
- **Cách tìm ra nguyên nhân:**
  1. Mở tab **Logs** trên Render dashboard để quan sát tiến trình khởi động container.
  2. Nhận thấy nền tảng cloud tự động cấp phát cổng động qua biến `$PORT` (ví dụ `PORT=10000`), nhưng nếu CMD trong Dockerfile ghim cứng `--port 8000` thì load balancer của platform sẽ không thể kết nối tới ứng dụng.
  3. Đồng thời, biến bí mật `AGENT_API_KEY` được khai báo `sync: false` trong `render.yaml` yêu cầu người vận hành phải nhập thủ công trên Render dashboard lúc tạo Blueprint. Nếu chưa điền hoặc điền sai ký tự, endpoint `/ask` sẽ lập tức trả về `401`.
- **Cách sửa:**
  1. Đảm bảo lệnh khởi chạy CMD trong `Dockerfile` sử dụng `${PORT:-8000}` để nhận diện cổng động:
     `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`
  2. Vào mục **Environment** của service `day12-agent` trên Render dashboard, kiểm tra và dán đúng giá trị `AGENT_API_KEY`, đồng thời kết nối đúng biến `REDIS_URL` từ service `day12-redis`.
  3. Nhấn **Save Changes** và trigger **Manual Deploy** ➔ Service khởi động thành công, endpoint `/health` trả về `200 OK` và `/ready` kết nối thành công với Redis.
