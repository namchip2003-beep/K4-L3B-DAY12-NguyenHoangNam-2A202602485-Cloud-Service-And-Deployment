# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyen Hoang Nam  Mã học viên: 2A202602485

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống: Deploy nhầm mà quên cấu hình key trên Cloud. Nếu dùng mặc định "changeme", ứng dụng vẫn chạy bình thường nhưng ai cũng có thể đoán được key và gọi API, dẫn đến việc bị lạm dụng tốn phí khổng lồ trước khi bạn nhận ra sự cố. "Chết sớm" khiến Healthcheck báo lỗi ngay và code rác không kịp chạy.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log: `{"event": "ask_completed", "level": "INFO", "timestamp": "2026-09-29T10:30:00Z", "user_id": "sv-test", "tokens_in": 15, "tokens_out": 42, "cost_usd": 0.001}`
> Hai việc làm được: 1) Gửi log này vào Elasticsearch/DataDog để vẽ biểu đồ tổng chi phí (sum of cost_usd) theo từng user_id. 2) Lọc và tìm kiếm nhanh những request có số tokens_out vượt quá ngưỡng nào đó mà không cần viết regex phức tạp để móc tách text như lệnh print.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | ~1010 MB |
| Multi-stage | ~156 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch đó chính là bộ thư viện chuẩn của hệ điều hành, trình biên dịch (C/C++ compiler), và những file rác sinh ra trong quá trình pip install đã bị bỏ đi khi dùng base image `slim` và kỹ thuật Multi-stage. Image production chỉ giữ lại file thực thi cần thiết.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Các layer cài đặt thư viện (`RUN pip install`) sẽ được lấy từ cache, chỉ những layer từ `COPY . .` trở xuống mới phải chạy lại. Nếu đặt `COPY . .` lên trước `RUN pip install`, Docker sẽ thấy code thay đổi nên nó hủy bỏ cache của lệnh cài đặt, dẫn đến việc mỗi lần sửa code nó đều phải tải và cài lại toàn bộ thư viện (rất chậm).

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: Hacker khai thác lỗ hổng (ví dụ RCE) -> Chạy được shell trong container -> Vì app chạy bằng root, hacker có quyền chèn mã độc, tải phần mềm bẻ khóa, hoặc lợi dụng cấu hình lỗi để leo thang quyền ra máy chủ vật lý bên ngoài (host). Lệnh `USER appuser` cắt đứt điều này: hacker chỉ chiếm được một user bị hạn chế, không có quyền tải hay thay đổi file hệ thống.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Có thể gửi tối đa 20 request trong 2 giây liên tiếp. Người dùng có thể spam 10 request vào lúc 10:00:59, rồi 1 giây sau (lúc 10:01:00) biến đếm bị reset, người đó spam tiếp 10 request nữa. Kết quả là tạo ra một cú sốc (spike) 20 request chỉ trong 2 giây mà hệ thống vẫn hợp lệ hóa. Sliding window chặn được mánh khóe này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Khác biệt: Rate limit đếm "số lần", Cost guard đếm "số tiền". 
> Tình huống 1: User mới gọi 1 request (chưa quá 10 lần -> Rate limit cho qua), nhưng câu lệnh đó bắt AI đọc cuốn sách dài tốn $20, trong khi ngân sách là $10 -> Cost guard sẽ chặn. 
> Tình huống 2: User hỏi 20 câu cực ngắn tốn 0.01$ (Cost guard cho qua), nhưng đã vượt mốc 10 câu/phút -> Rate limit báo lỗi 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Sự kiện: 
> 1) Redis sập 30s. 
> 2) Cloud gọi `/health` và nhận lỗi vì Redis không phản hồi. 
> 3) Orchestrator lầm tưởng cả container đã hỏng nên lập tức gửi tín hiệu giết process. 
> 4) Cả 3 container bị restart cùng lúc -> Toàn bộ ứng dụng sập (Downtime). 
> Tách riêng giúp `/ready` chặn bớt traffic, còn `/health` báo "tôi vẫn đang đợi Redis" để khỏi bị giết oan.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Nếu lưu trong RAM bằng dict Python, `history_length` sẽ nhảy lung tung (1, 2, 1, 3, 2, 1...). Lý do là các yêu cầu liên tiếp có thể được Load balancer phân bổ ngẫu nhiên vào 1 trong 3 container khác nhau. Vì chúng có "trí nhớ" riêng lẻ, con số này không bao giờ đồng bộ. Khi đưa lên Redis, độ dài lịch sử sẽ tăng dần ổn định.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Thông báo lỗi: `Attempt #1 failed with service unavailable. Continuing to retry` ở bước Healthcheck trên Railway.
> Nguyên nhân: Đọc log console thì thấy ứng dụng chạy bình thường nhưng fail ở lệnh Pydantic báo thiếu biến môi trường, dẫn tới crash ngay khi vừa bật lên.
> Cách sửa: Truy cập vào trang Dashboard của Railway -> qua thẻ Variables -> Add Variable `AGENT_API_KEY`, `RATE_LIMIT_PER_MINUTE`, `MONTHLY_BUDGET_USD`... và điền các tham số tương ứng. Sau khi lưu, ứng dụng khởi động lại và Healthcheck báo thành công.
