# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng giữ chỗ bên dưới bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Chí Hướng  Mã học viên: 2A202601203

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Một tình huống cụ thể là lúc deploy service `agent` lên Railway nhưng quên
> đặt `AGENT_API_KEY`. Vì trường này không có mặc định, Pydantic sẽ báo lỗi ngay
> khi service khởi động và deployment không qua healthcheck. Nhờ vậy tôi biết
> cấu hình đang thiếu trước khi public URL nhận traffic. Nếu dùng khóa mặc định
> `"changeme"`, service vẫn chạy và người lạ có thể đoán khóa để gọi `/ask`, làm
> phát sinh chi phí mà tôi không phát hiện ngay.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log thu được từ bản chạy thật là:
>
> ```json
> {"event":"ask_completed","level":"info","timestamp":"2026-08-10T04:40:48.929256+00:00","user_id":"exercise-log","tokens_in":6,"tokens_out":40,"cost_usd":0.0000249}
> ```
>
> Với log có cấu trúc này, tôi có thể lọc tất cả event `ask_completed` của một
> `user_id` để điều tra hoạt động của user đó. Tôi cũng có thể cộng
> `cost_usd`, `tokens_in` và `tokens_out` để theo dõi chi phí hoặc tạo cảnh báo.
> Dòng `print("đã trả lời xong")` không có trường dữ liệu ổn định để máy lọc,
> nhóm hoặc tính toán như vậy.

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
| 1 stage (bản đầu) | 1.73 GB |
| Multi-stage | 270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build lại đúng Dockerfile 1-stage ban đầu từ lịch sử Git và đo được
> `agent:single` là 1.73 GB, còn `agent:multi` là 270 MB. Phần chênh lệch chủ
> yếu đến từ base image `python:3.11` đầy đủ so với `python:3.11-slim` và việc
> stage runtime chỉ nhận dependency đã cài từ builder. Nó không mang toàn bộ
> môi trường build và những thành phần hệ điều hành không cần thiết sang image
> chạy production.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, các layer copy `requirements.txt` và chạy
> `pip install` ở builder vẫn dùng cache vì dependency không đổi. Những layer
> runtime trước khi copy source, như tạo `appuser` và copy `/install`, cũng được
> dùng lại. Layer `COPY app ./app` và các layer sau nó phải tạo lại. Nếu đặt
> `COPY . .` trước `RUN pip install`, chỉ một thay đổi trong source cũng làm
> layer copy đổi và buộc Docker cài lại toàn bộ thư viện, nên build chậm hơn rõ
> rệt.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu ứng dụng Python có lỗ hổng cho phép chạy lệnh, kẻ tấn công trước hết có
> quyền của process trong container. Khi process chạy root, họ có toàn quyền
> bên trong container; nếu tiếp tục khai thác được lỗi container escape, Docker
> socket hoặc một host volume cấu hình nguy hiểm thì mức ảnh hưởng lên host rất
> lớn. Lệnh `USER appuser` chuyển process sang UID 10001 không đặc quyền, nên
> ngay từ bước chạy lệnh trong container, kẻ tấn công chỉ nhận quyền hạn chế.
> Đây không thay thế sandbox nhưng làm giảm đáng kể phạm vi thiệt hại.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi tối đa 20 request trong khoảng 2 giây. Họ gửi 10
> request ở cuối phút, ví dụ 10:00:59, rồi gửi tiếp 10 request ngay đầu phút kế
> tiếp, ví dụ 10:01:00. Bộ đếm theo phút đã reset nên cả hai nhóm đều hợp lệ.
> Sliding window 60 giây vẫn nhìn thấy nhóm request trước nên sẽ chặn nhóm sau
> khi tổng số trong 60 giây đạt hạn mức.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn số request trong một khoảng thời gian ngắn, còn cost guard
> giới hạn tổng số tiền một user được tiêu trong tháng. Ví dụ user chỉ gửi một
> request trong phút nhưng đã gần hết ngân sách tháng hoặc gửi prompt rất đắt:
> rate limit cho qua nhưng cost guard phải chặn. Ngược lại, user gửi 11 request
> rất ngắn trong 60 giây khi ngân sách tháng vẫn còn nhiều: cost guard vẫn cho
> phép về mặt tiền, nhưng rate limiter phải trả 429 vì gọi quá nhanh.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp `/health` với `/ready`, khi Redis mất kết nối thì cả ba container đều
> trả 503 cho liveness probe dù process Python vẫn sống. Orchestrator hiểu nhầm
> cả ba process bị hỏng và restart chúng gần như cùng lúc. Các instance đang xử
> lý request bị cắt, cụm mất khả năng phục vụ, rồi có thể tiếp tục restart vòng
> lặp nếu Redis chưa hồi phục. Tách riêng giúp `/health` vẫn báo process sống,
> còn `/ready` chỉ yêu cầu load balancer tạm ngừng gửi traffic vào instance.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis dùng chung, lần gọi đầu tiên có `history_length = 0`, lần thứ hai
> thấy hai message trước đó nên có `history_length = 2`; các lần sau tiếp tục
> tăng theo cặp user/assistant dù request đi vào instance khác. Nếu dùng một
> dict Python trong từng container, mỗi instance có lịch sử riêng. Khi load
> balancer đổi instance, giá trị sẽ lúc tăng, lúc quay về 0 hoặc một giá trị
> thấp hơn, khiến agent có biểu hiện mất trí nhớ không ổn định.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi thật tôi gặp ở lần deploy Railway đầu tiên là
> `Invalid value for '--port': '$PORT' is not a valid integer`. Tôi thấy thông
> báo này trong deploy log sau khi build thành công nhưng healthcheck thất bại.
> Nguyên nhân là `startCommand` trong `railway.toml` truyền nguyên chuỗi `$PORT`
> cho Uvicorn thay vì mở rộng biến môi trường. Tôi xóa override `startCommand`
> để Railway dùng `CMD` trong Dockerfile; lệnh đó chạy qua `sh -c` và đọc
> `${PORT:-8000}` đúng cách. Deployment kế tiếp chuyển sang `SUCCESS`, `/health`
> và `/ready` đều trả 200.
