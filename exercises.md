# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng giữ chỗ bên dưới bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lê Trọng Khánh  Mã học viên: 2A202602941

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Ví dụ cụ thể là lúc deploy lên Railway nhưng quên đặt `AGENT_API_KEY`. Nếu
> ứng dụng dùng mặc định `"changeme"`, container vẫn báo healthy và endpoint
> `/ask` bị bảo vệ bằng một khóa mà ai đọc source cũng biết; mình chỉ phát hiện
> sau khi service đã public và có thể đã phát sinh chi phí. Fail fast làm
> process dừng ngay ở bước khởi động, log chỉ rõ thiếu cấu hình và deployment
> không được nhận traffic, nên lỗi được sửa trước khi trở thành sự cố bảo mật.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log thật mình lấy từ service Railway là:
>
> ```json
> {"event":"ask_completed","level":"info","tokens_in":92,"user_id":"cp5-test","tokens_out":45,"cost_usd":0.0000408,"timestamp":"2026-09-29T04:27:10.054563+00:00"}
> ```
>
> Với JSON, mình có thể (1) lọc và nhóm theo `user_id`, `event` hoặc khoảng
> thời gian để tìm request lỗi/chậm, và (2) cộng `tokens_in`, `tokens_out`,
> `cost_usd` để làm dashboard hoặc cảnh báo chi phí. Chuỗi
> `print("đã trả lời xong")` không có trường dữ liệu ổn định để máy truy vấn hay
> tổng hợp.

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
| 1 stage (bản đầu) | khoảng 1120 MB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Mình đo bản multi-stage hiện tại bằng Docker Desktop được 271 MB. Bản
> single-stage dùng image `python:3.11` đầy đủ vào khoảng 1120 MB (con số có
> thể lệch nhẹ theo phiên bản base image và kiến trúc). Phần chênh lệch chủ yếu
> là hệ điều hành đầy đủ, công cụ build và cache cài package có trong image
> single-stage. Bản runtime chỉ dùng `python:3.11-slim` và copy dependency đã
> cài từ stage `builder`, nên không mang toàn bộ môi trường build sang production.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, các layer base image, `ENV`, tạo user, `WORKDIR`,
> `COPY requirements.txt` và `RUN pip install` vẫn lấy từ cache. Layer
> `COPY . .` chứa source bị invalid và các instruction đứng sau nó được xét
> lại; các instruction metadata như `USER`, `EXPOSE`, `HEALTHCHECK`, `CMD`
> gần như không tốn thời gian. Nếu đặt `COPY . .` trước `RUN pip install`, bất
> kỳ thay đổi source nào cũng làm layer copy đổi, kéo theo việc cài lại toàn bộ
> dependency dù `requirements.txt` không đổi, khiến build chậm hơn rõ rệt.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu code Python có lỗi cho phép thực thi lệnh, kẻ tấn công trước hết chạy
> lệnh với UID của process trong container. Khi process là root, họ có toàn
> quyền trong container; nếu host còn mount Docker socket/thư mục nhạy cảm hoặc
> có lỗ hổng container runtime/kernel, quyền đó có thể được dùng để sửa dữ liệu
> host hay thoát container với quyền cao. `USER app` cắt chuỗi ngay sau bước
> thực thi mã: payload chỉ có quyền của user thường, không thể tùy ý sửa file hệ
> thống hay dùng các thao tác đặc quyền trong container. Nó không thay thế việc
> vá lỗ hổng, nhưng giảm mạnh hậu quả nếu app bị khai thác.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa là **20 request trong 2 giây**. Người dùng gửi 10 request ngay trước
> ranh giới phút, ví dụ từ `12:00:59.0` đến `12:00:59.9`, rồi gửi tiếp 10
> request ngay sau khi bộ đếm reset ở `12:01:00`. Fixed window coi đây là hai
> phút khác nhau nên cho qua cả hai nhóm, dù nhìn theo bất kỳ cửa sổ trượt 2
> giây nào thì đó là một burst 20 request.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn **tần suất** trong cửa sổ ngắn để chống burst và bảo vệ
> tài nguyên tức thời; cost guard giới hạn **tổng tiền** tích lũy theo user trong
> tháng. Một user chỉ gọi vài request thưa nhau nên rate limit cho qua, nhưng đã
> dùng hết ngân sách tháng thì cost guard phải trả 402. Ngược lại, user chưa
> tiêu đáng kể nhưng gửi request thứ 11 trong vòng 60 giây thì cost guard vẫn
> cho phép về mặt ngân sách, còn rate limiter phải trả 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự xảy ra sẽ là: Redis mất kết nối → endpoint gộp của cả ba container
> cùng trả lỗi → orchestrator hiểu nhầm rằng cả ba process đã chết → nó restart
> đồng loạt các container → container mới vẫn không kết nối được Redis nên lại
> fail health check → hình thành vòng lặp restart cho tới khi Redis phục hồi.
> Việc restart còn làm rớt request đang xử lý và tạo thêm tải lúc sự cố. Tách
> hai probe tránh chuyện này: `/health` vẫn 200 vì process còn sống, còn
> `/ready` trả 503 để load balancer tạm ngừng gửi traffic mà không restart app.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis dùng chung, dù request được phân phối vào instance nào thì
> `history_length` vẫn tăng đều theo số message đã lưu, ví dụ `0, 2, 4, 6...`
> vì mỗi lượt hỏi ghi một message user và một message assistant. Nếu dùng dict
> Python, mỗi container có một lịch sử riêng: request vào A có thể thấy 2, request
> kế tiếp vào B lại thấy 0, rồi quay lại A thấy 4. Con số sẽ nhảy lên xuống tùy
> load balancer và trở về 0 khi container restart, biểu hiện service còn state
> cục bộ và không scale an toàn.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi mình gặp sau khi Railway báo deployment `SUCCESS` là mọi request public
> vẫn trả `502 Application failed to respond`. Mình đọc runtime log và thấy
> Uvicorn đang nghe ở `0.0.0.0:8080`, trong khi domain Railway lúc tạo ban đầu
> lại được route tới port 8000. Mình cập nhật target port của domain thành 8080,
> chờ DNS/domain đồng bộ rồi gọi lại. Kết quả sau sửa là `/health` trả 200,
> `/ready` trả 200 với `redis: true`, `/ask` thiếu key trả 401 và có key trả 200.
