# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `Câu trả lời` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Doãn Hoàng  Mã học viên: 2A202601119

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu quên cấu hình AGENT_API_KEY trên Cloud, việc ứng dụng báo lỗi và dừng chạy ngay lập tức lúc khởi động (Fail Fast) giúp chúng ta phát hiện ngay là thiếu biến cấu hình để bổ sung. Nếu để mặc định là "changeme", ứng dụng vẫn khởi động thành công và kẻ tấn công có thể dò ra khóa mặc định để gọi API LLM miễn phí, gây tổn thất lớn về chi phí trước khi chúng ta phát hiện.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON: `{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T03:47:08.123456+00:00", "user_id": "sv-test", "cost_usd": 0.0001}`
> Hai việc làm được:
> 1. Dễ dàng dùng các công cụ quản lý log (Datadog, ElasticSearch) để lọc và thống kê tổng chi phí `cost_usd` của từng `user_id` theo thời gian.
> 2. Thiết lập hệ thống cảnh báo tự động khi chi phí tăng đột ngột dựa trên các trường `cost_usd` và `level`.

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
| 1 stage (bản đầu) | ~1 GB |
| Multi-stage | ~180 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch (~800MB) bao gồm các công cụ biên dịch (compilers), dev headers, pip install cache và base image Debian đầy đủ ban đầu. Bản multi-stage chỉ copy các compiled packages sang runtime stage dùng base image slim nhẹ nên tối ưu được tối đa dung lượng.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi sửa một ký tự trong app/main.py và build lại: các layer chứa requirements.txt và pip install sẽ được dùng lại từ cache, chỉ có layer COPY . . và các layer tiếp theo phải chạy lại. Nếu đặt COPY . . trước pip install, mỗi lần sửa code sẽ làm mất hiệu lực cache của layer COPY, khiến Docker phải cài lại toàn bộ thư viện từ đầu, làm tăng thời gian build.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu container chạy root và code Python có lỗ hổng thực thi lệnh hệ thống (RCE), kẻ tấn công sẽ có quyền root trong container. Từ đó, họ có thể khai thác các lỗ hổng nhân (kernel vulnerability) để thoát khỏi container và chiếm quyền root của máy host. Lệnh USER appuser giới hạn quyền chạy của tiến trình trong container chỉ là user thường, cắt đứt khả năng thực hiện các lệnh nguy hiểm ngay cả khi code bị hack.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi tối đa 20 request trong 2 giây liên tiếp. Ví dụ: gửi 10 request ở giây 10:00:59 (cuối phút thứ nhất) và gửi tiếp 10 request ở giây 10:01:01 (đầu phút thứ hai). Vì đếm theo phút đồng hồ reset lúc giây 00 nên cả hai lượt gửi đều nằm trong hạn mức 10/phút của phút đó, nhưng thực tế đã spam 20 request trong 2 giây.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn số lượng request trong một khoảng thời gian ngắn để chống spam, còn Cost guard giới hạn tổng số tiền để kiểm soát ngân sách LLM. Tình huống 1: User gửi 1 request/phút nhưng request đó chứa input cực dài (đốt sạch 100k tokens), Rate limit cho qua nhưng Cost guard chặn vì hết ngân sách. Tình huống 2: User gửi 50 request trong 10 giây nhưng mỗi request rất ngắn (tiêu tốn rất ít tiền), Cost guard cho qua nhưng Rate limit chặn do tần suất quá cao.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Khi Redis mất kết nối 30 giây:
> 1. Cụm 3 container đều báo unhealthy qua endpoint gộp.
> 2. Orchestrator coi cả 3 container đã chết và tiến hành restart tất cả cùng lúc.
> 3. Trong quá trình khởi động lại, không có container nào sẵn sàng phục vụ và nếu Redis chưa kịp sống lại thì chúng lại tiếp tục bị restart vòng lặp, dẫn đến sập toàn hệ thống.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Do load balancer phân phối các request xoay vòng giữa 3 container, câu hỏi 1 đi vào container A (RAM của A), câu hỏi 2 đi vào container B. Vì RAM của B không có dữ liệu của A nên user_id đó sẽ thấy history_length báo nhảy lung tung (hoặc về 0), Agent bị "mất trí nhớ" ngẫu nhiên giữa các câu hỏi.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi gặp phải: Deployment bị báo lỗi Network > Healthcheck failure và Deploy Logs ghi nhận "Error: Invalid value for '--port': '$PORT' is not a valid integer".
> Cách tìm nguyên nhân: Đọc Deploy Logs trên dashboard của Railway.
> Cách sửa: Sửa startCommand trong railway.toml thành "sh -c 'uvicorn app.main:app --host 0.0.0.0 --port $PORT'" để biến môi trường $PORT được shell dịch thành số nguyên chính xác.
