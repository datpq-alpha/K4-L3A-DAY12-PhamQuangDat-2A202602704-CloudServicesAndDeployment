# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Phạm Quang Đạt  Mã học viên: 2A202602704

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy service lên Railway, tôi quên cấu hình biến AGENT_API_KEY. Vì trường này không có giá trị mặc định, ứng dụng thất bại ngay khi khởi động và dashboard báo lỗi, nên tôi phát hiện và bổ sung secret trước khi service nhận traffic. Nếu code dùng mặc định "changeme", ứng dụng vẫn hoạt động với một khóa dễ đoán; người khác có thể gọi /ask trái phép, làm phát sinh chi phí LLM hoặc sử dụng hết ngân sách trước khi tôi nhận ra.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thực tế:
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T10:37:38.067224+00:00", "user_id": "sv01", "tokens_in": 48, "tokens_out": 47, "cost_usd": 3.54e-05}  
Từ log JSON này, tôi có thể:  1) Lọc và truy vết request theo event, user_id và timestamp, giúp xác định người dùng nào gọi API và thời điểm xảy ra sự kiện. 2) Tổng hợp tokens_in, tokens_out, cost_usd để theo dõi chi phí và thiết lập cảnh báo khi mức sử dụng tăng bất thường.  
Còn lệnh print("đã trả lời xong") chỉ chứa một đoạn văn bản chung, không có các trường dữ liệu có cấu trúc để lọc, thống kê hoặc tạo cảnh báo tự động.

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
| 1 stage (bản đầu) | 1.6 GB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Image single-stage lớn hơn vì dùng base image python:3.11 đầy đủ, chứa nhiều package hệ điều hành và công cụ build không cần thiết khi chạy ứng dụng. Ngoài ra, dependency được cài trực tiếp trong image chạy thật nên pip cache và các tệp trung gian cũng có thể bị giữ lại.
Image multi-stage dùng python:3.11-slim. Dependency được cài tại stage builder, sau đó stage runtime chỉ nhận các package đã cài cùng source code cần thiết. Vì vậy image không mang theo môi trường build và các tệp thừa, giúp giảm dung lượng, tải/deploy nhanh hơn và giảm bề mặt tấn công.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile hiện tại, khi tôi chỉ sửa `app/main.py`, toàn bộ stage `builder`
> vẫn dùng cache: base image, `WORKDIR`, `COPY requirements.txt` và
> `RUN pip install` không thay đổi. Ở stage runtime, các layer đến hết
> `COPY --from=builder /install /usr/local` cũng được dùng lại. Cache bắt đầu mất tại
> `COPY app ./app`; các layer đứng sau nó như `COPY utils ./utils`, `USER`,
> `EXPOSE`, `HEALTHCHECK` và `CMD` phải được tạo lại, dù phần lớn rất nhẹ. Nếu
> đặt `COPY . .` trước `RUN pip install`, chỉ cần thay một ký tự trong source cũng
> làm layer `COPY` đổi và buộc bước cài dependency chạy lại. Build vì thế chậm
> hơn và không tận dụng được cache dependency khi `requirements.txt` vẫn giữ
> nguyên.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu ứng dụng Python có lỗ hổng cho phép thực thi mã từ xa, kẻ tấn công có thể
> chạy lệnh với đúng user của process trong container. Nếu process chạy bằng
> root, chúng có quyền root trong container: có thể sửa file hệ thống, đọc secret
> hoặc dữ liệu được mount, cài công cụ và lợi dụng thêm cấu hình nguy hiểm như
> Docker socket, capability dư thừa, privileged mode hoặc một lỗ hổng kernel để
> tác động tới host. Lệnh `USER appuser` cắt chuỗi này ngay sau bước chiếm quyền
> thực thi mã: mã độc chỉ chạy với UID thường, không tự động có quyền sửa tài
> nguyên dành cho root. Đây là một lớp giảm thiểu thiệt hại; nó không thay thế
> việc bỏ privileged mode, giới hạn capability và không mount Docker socket.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi tối đa 20 request trong 2 giây. Họ gửi 10 request ở
> cuối phút cũ, ví dụ từ `12:00:59` đến ngay trước `12:01:00`, rồi gửi thêm 10
> request ngay sau `12:01:00`. Bộ đếm theo phút đồng hồ reset ở giây 00 nên hai
> nhóm được tính vào hai cửa sổ khác nhau, dù trên thực tế cả 20 request nằm
> trong khoảng thời gian chỉ 2 giây. Sliding window 60 giây tránh được cú burst
> ở ranh giới này vì tại thời điểm nhóm thứ hai đến, 10 request trước vẫn còn
> nằm trong cửa sổ đang xét.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit kiểm soát tần suất request trong một khoảng thời gian ngắn, còn
> cost guard kiểm soát tổng chi phí tích lũy của từng user trong tháng. Rate
> limit có thể cho qua request đầu tiên của phút nhưng cost guard vẫn phải chặn
> nếu user đã dùng gần hết ngân sách và chi phí ước tính của request mới làm
> tổng vượt mức tháng. Ngược lại, khi ngân sách tháng vẫn còn nhiều, một user
> gửi liên tiếp 11 request rẻ trong vòng 60 giây sẽ bị rate limit chặn ở request
> thứ 11, dù cost guard vẫn có thể cho phép vì tổng chi phí còn thấp.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu `/health` đồng thời kiểm tra Redis thì khi Redis mất kết nối, cả ba
> container lần lượt trả health check lỗi. Orchestrator đánh dấu chúng
> unhealthy và bắt đầu restart; vì cả ba phụ thuộc cùng một Redis, chúng có thể
> bị restart gần như đồng thời. Redis vẫn chưa trở lại nên các container mới
> tiếp tục fail health check, tạo vòng lặp restart và làm cụm không còn instance
> ổn định để phục vụ. Khi Redis hoạt động lại, container mới có thể vượt health
> check và nhận traffic, nhưng hệ thống đã chịu downtime và nhiều lần restart
> không cần thiết. Khi tách đúng hai endpoint, `/health` vẫn trả 200 vì process
> còn sống, còn `/ready` trả 503 để load balancer tạm ngừng gửi request. Redis
> phục hồi thì `/ready` tự trở lại 200 mà không cần giết cả ba container.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Khi chạy 3 instance và lưu lịch sử trong Redis, tôi quan sát `history_length`
> lần lượt là `0, 2, 4, 6, 8`. Dù request được phân phối tới các container khác
> nhau, tất cả đều đọc cùng dữ liệu trong Redis nên lịch sử vẫn tăng liên tục.
>
> Bằng chứng log cho thấy cả ba instance đã xử lý request:

> agent-2  | {"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T11:08:35.662621+00:00", 
"user_id": "q9-3068ea253190436aac04ed1a6d2b607d", "tokens_in": 43, "tokens_out": 44, "cost_usd": 
3.285e-05}
agent-2  | {"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T11:08:35.787368+00:00", 
"user_id": "q9-3068ea253190436aac04ed1a6d2b607d", "tokens_in": 179, "tokens_out": 50, "cost_usd": 
5.685e-05}
agent-1  | {"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T11:08:35.709071+00:00", 
"user_id": "q9-3068ea253190436aac04ed1a6d2b607d", "tokens_in": 88, "tokens_out": 44, "cost_usd": 
3.96e-05}
agent-3  | {"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T11:08:35.605912+00:00", 
"user_id": "q9-3068ea253190436aac04ed1a6d2b607d", "tokens_in": 1, "tokens_out": 40, "cost_usd": 
2.415e-05}
agent-3  | {"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T11:08:35.731347+00:00", 
"user_id": "q9-3068ea253190436aac04ed1a6d2b607d", "tokens_in": 134, "tokens_out": 44, "cost_usd": 
4.65e-05}  

> Nếu dùng dict Python, mỗi container sẽ có bộ nhớ riêng. Khi Nginx phân phối
> request lần lượt qua ba instance, kết quả có thể giống `0, 0, 0, 2, 2, 2`
> hoặc tăng không đều tùy instance nhận request, khiến người dùng có cảm giác
> agent bị mất lịch sử.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Trong lúc chuẩn bị deploy CP5 theo hướng Cloud Run kết hợp Upstash, lệnh kiểm
> tra kết nối Redis báo lỗi
> `redis.exceptions.AuthenticationError: invalid username-password pair`. Tôi
> đọc traceback và thấy chương trình đã kết nối tới máy chủ Upstash nhưng bị từ
> chối tại bước xác thực, nên nguyên nhân nằm ở TCP URL, username hoặc token
> không khớp chứ không phải code lưu trữ Redis. Do chưa kịp hoàn thiện cấu hình
> Google Cloud và `gcloud`, tôi bỏ connection Upstash bị lỗi và chuyển sang
> deploy bằng Render Blueprint. Blueprint tạo Render Key Value rồi tự gắn
> connection string vào biến `REDIS_URL` của `day12-agent`. Sau khi deploy, tôi
> gọi `/health` nhận 200 và `/ready` nhận 200 với `redis: true`, xác nhận service
> và Redis đã kết nối thành công.
