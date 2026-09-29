# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay các dòng mẫu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Hoàng Tuyên  Mã học viên: 2A202602439

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu đặt giá trị mặc định như `"changeme"`, khi deploy lên server staging hoặc production mà lỡ quên thiết lập biến môi trường `AGENT_API_KEY`, ứng dụng vẫn sẽ khởi động bình thường. Khi đó, API `/ask` mở ra public sẽ được bảo vệ bởi chìa khóa mặc định `"changeme"`. Bất kỳ kẻ quét bot hoặc người lạ nào đoán được mật khẩu mặc định đều có thể gọi vào hệ thống và tiêu tốn toàn bộ chi phí token LLM của chúng ta mà ta không hề hay biết cho đến khi hóa đơn đội giá. Ngược lại, việc "fail-fast" (không có giá trị mặc định) khiến app crash ngay lập tức tại thời điểm deploy kèm thông báo `ValidationError: Field required`, buộc lập trình viên phải cấu hình đúng secret ngay lúc đó trước khi phục vụ bất kỳ traffic nào.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON thu được:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T10:24:00.123456+00:00", "user_id": "sv-test", "tokens_in": 12, "tokens_out": 25, "cost_usd": 0.000185}`
>
> Hai việc làm được:
> 1. Truy vấn, lọc và tổng hợp có cấu trúc bằng các hệ thống quản trị log tập trung (như Datadog, CloudWatch, Elasticsearch, Grafana Loki): ví dụ tính tổng `cost_usd` theo từng `user_id` trong ngày, hoặc thống kê P95 lượng token tiêu thụ.
> 2. Thiết lập hệ thống cảnh báo tự động (alerting) dựa trên điều kiện ngưỡng: ví dụ tự động bắn cảnh báo qua Slack nếu `cost_usd > 0.01` trên một request hoặc số lượng log có `level == "error"` vượt ngưỡng cho phép trong 5 phút. `print()` thô không thể parse trường và trường dữ liệu tự động như vậy.

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
| 1 stage (bản đầu) | 1020 MB |
| Multi-stage | 185 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch (~835 MB) chủ yếu là:
> 1. Bộ công cụ biên dịch mã nguồn (gcc, g++, make, build-essential) và các gói thư viện C phát triển (dev headers) cần thiết để biên dịch dependency nhưng hoàn toàn không cần thiết khi chạy ứng dụng.
> 2. Package manager cache (apt cache), tài liệu hướng dẫn (manpages), và các tiện ích hệ điều hành Debian đầy đủ vốn có trong base image `python:3.11` nhưng đã được lược bỏ tối đa trong bản `python:3.11-slim`.
> 3. Trong multi-stage, stage builder chịu trách nhiệm cài đặt và biên dịch, sau đó stage runtime chỉ sao chép kết quả đã cài đặt tại `/install` sang môi trường runtime sạch.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> - Với Dockerfile hiện tại: Các layer từ đầu cho tới `COPY requirements.txt .` và `RUN pip install ...` ở stage builder, cũng như việc copy `/install` ở runtime đều được dùng lại hoàn toàn từ cache (CACHE HIT) vì `requirements.txt` không thay đổi. Chỉ có layer `COPY app ./app` và các bước kế tiếp bị invalidate cache và phải chạy lại, giúp việc build chỉ mất khoảng 1-2 giây.
> - Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi lần sửa bất kỳ ký tự nào trong code, layer `COPY . .` sẽ bị đổi hash và làm toàn bộ các layer phía sau mất cache (cache bust). Docker sẽ phải chạy lại lệnh `pip install` từ đầu, tải và cài đặt lại toàn bộ thư viện mỗi lần sửa code, làm chậm nghiêm trọng quy trình phát triển và deploy.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện dẫn tới quyền cao trên host:
> 1. Kẻ tấn công phát hiện và khai thác một lỗ hổng trong code Python (ví dụ: Remote Code Execution, command injection hoặc arbitrary file write).
> 2. Vì container chạy mặc định bằng root (UID 0), tiến trình bị kẻ tấn công chiếm quyền điều khiển cũng sở hữu quyền root (UID 0) bên trong container.
> 3. Kẻ tấn công lợi dụng các lỗ hổng container escape (như lỗi kernel Linux, lỗ hổng runc/containerd, hoặc volume mount nhầm `/var/run/docker.sock`, thư mục hệ thống của host).
> 4. Do UID 0 trong container mặc định ánh xạ trực tiếp với UID 0 (root) trên máy host (nếu không dùng user namespace), kẻ tấn công thoát ra khỏi container và lập tức sở hữu đặc quyền root tối cao trên toàn bộ máy host.
>
> Lệnh `USER appuser` (UID 10001) cắt đứt chuỗi này ngay tại bước 2: Tiến trình Python chạy với quyền người dùng không đặc quyền. Khi bị chiếm quyền, kẻ tấn công chỉ có quyền hạn chế trong sandbox container, không thể ghi vào thư mục hệ thống container và không thể kích hoạt các vector tấn công breakout đòi hỏi đặc quyền root.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.
> Cách đạt được:
> - Người dùng gửi 10 request vào đúng giây thứ 59 của phút trước (10:00:59). Hệ thống fixed window ghi nhận 10/10 request cho phút 10:00 và cho phép.
> - Đúng 1 giây sau (10:01:00), đồng hồ bước sang phút mới và bộ đếm tự động reset về 0.
> - Người dùng lập tức gửi tiếp 10 request vào giây 10:01:00 (hoặc 10:01:01). Cả 10 request này lại được duyệt vì tính cho phút 10:01.
> Kết quả: Trong khoảng thời gian chỉ 2 giây (10:00:59 - 10:01:01), hệ thống đã phải chịu tải 20 request, gấp đôi hạn mức mong muốn. Sliding window giải quyết triệt để lỗi này bằng cách luôn xét đúng khoảng thời gian trôi 60 giây gần nhất tính từ thời điểm gọi.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> - Điểm khác nhau: Rate limit kiểm soát tần suất request trong ngắn hạn (tính bằng số lượng request/phút) để ngăn ngừa quá tải hạ tầng và chống DoS. Cost guard kiểm soát chi phí tài chính tích lũy trong dài hạn (tính bằng tổng số tiền USD/tháng) để ngăn chặn việc cháy ngân sách API LLM.
> - Tình huống Rate limit cho qua nhưng Cost guard phải chặn: Người dùng chỉ gửi 1 request mỗi 10 phút (tần suất cực thấp, hoàn toàn dưới hạn mức 10 req/phút), nhưng mỗi request gửi đoạn văn bản khổng lồ tiêu thụ 100.000 tokens (tốn ~1.5 USD). Sau vài lượt, user đã tiêu hết sạch ngân sách 10.0 USD của tháng; lúc này dù cả ngày user mới gửi 1 request thì Cost guard vẫn lập tức chặn lại và trả về 402 Payment Required.
> - Tình huống Cost guard cho qua nhưng Rate limit phải chặn: Đầu tháng tài khoản user chưa tiêu đồng nào (ngân sách 10.0 USD còn nguyên), nhưng một script lỗi gửi dồn dập 20 request chỉ trong 3 giây (mỗi request chỉ tiêu tốn 0.0001 USD). Cost guard không chặn vì còn nhiều tiền, nhưng Rate limit sẽ chặn từ request thứ 11 và trả về 429 Too Many Requests để bảo vệ server khỏi bị nghẽn.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện xảy ra:
> 1. Redis gặp sự cố mạng hoặc khởi động lại, mất kết nối trong 30 giây.
> 2. Cơ chế Liveness Probe của orchestrator (Kubernetes / Docker) định kỳ gọi vào endpoint kiểm tra sức khỏe của cả 3 container agent.
> 3. Do endpoint này kiểm tra kết nối Redis và Redis đang chết, cả 3 container agent đều phản hồi thất bại (503 Service Unavailable).
> 4. Orchestrator suy luận rằng bản thân cả 3 tiến trình container agent đều đã bị treo hoặc chết hỏng, nên kích hoạt lệnh KILL và RESTART toàn bộ cả 3 container cùng lúc.
> 5. Khi Redis vừa hồi phục sau 30 giây, 3 container agent vẫn đang trong quá trình bị tắt hoặc đang khởi động lại từ đầu, dẫn tới việc không còn bất kỳ container nào phục vụ traffic -> toàn bộ hệ thống sập hoàn toàn (cascading failure).
> (Nếu tách biệt: `/health` chỉ kiểm tra process sống -> không bị restart; `/ready` kiểm tra Redis -> Load Balancer chỉ tạm thời ngưng đẩy traffic vào cho đến khi Redis sống lại).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> - Khi lưu trong Redis (Stateless): Toàn bộ 3 container agent đều chia sẻ chung một cơ sở dữ liệu Redis tập trung tại key `history:user_id`. Bất kể Load Balancer định tuyến câu hỏi tiếp theo vào container nào, container đó đều đọc được toàn bộ hội thoại trước đó, và `history_length` tăng đều đặn theo mỗi lượt: 0 -> 2 -> 4 -> 6...
> - Nếu lưu trong dict Python (Stateful trong RAM): Mỗi container chỉ lưu trữ dữ liệu trong bộ nhớ riêng của nó. Khi gọi 3 lượt liên tiếp, do request được phân phối xoay vòng (Round Robin) tới các container A, B, C, mỗi container chỉ thấy lượt đầu tiên gửi tới nó, khiến `history_length` bị nhảy thất thường (ví dụ: lượt 1 vào A báo 0, lượt 2 vào B báo 0, lượt 3 vào C báo 0, lượt 4 vào A mới thấy báo 2). Agent bị hiện tượng "mất trí nhớ ngắt quãng" giữa các container.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> - Lỗi gặp phải: Health check timeout / Web service failed to bind port trên nền tảng Cloud.
> - Thông báo lỗi: `Container failed to listen on port 8000. Health check probe timed out.`
> - Nguyên nhân: Các nền tảng Cloud (Render, Railway, Cloud Run) tự động cấp phát một cổng động thông qua biến môi trường `$PORT` (ví dụ `PORT=10000`). Nếu Dockerfile cố định chạy `--port 8000` và `--host 127.0.0.1`, ứng dụng sẽ chỉ lắng nghe trên cổng 8000 và chỉ chấp nhận kết nối nội bộ, khiến bộ thăm dò sức khỏe của nền tảng không thể kết nối tới ứng dụng.
> - Cách sửa: Trong `Dockerfile`, cấu hình lệnh chạy sử dụng cú pháp shell để đọc biến môi trường `$PORT`:
>   `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`
>   Lệnh này giúp ứng dụng vừa mở ra toàn bộ network interface (`0.0.0.0`), vừa ưu tiên cổng do Cloud chỉ định qua `$PORT`, và tự động fallback về cổng 8000 khi chạy trên máy cục bộ.
