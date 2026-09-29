# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng mẫu dưới mỗi câu hỏi bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Gia Khánh  Mã học viên: 2A202602851

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi quên cấu hình `AGENT_API_KEY` trên cloud, app dừng ngay trong lúc khởi động nên lần deploy bị báo lỗi và tôi biết để sửa trước khi service nhận traffic. Nếu dùng mặc định `"changeme"`, app vẫn khởi động bình thường; người ngoài có thể đoán key này và gọi API, làm phát sinh request và chi phí ngoài ý muốn.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> `{"event":"ask_completed","level":"info","timestamp":"2026-09-29T05:06:51.815166+00:00","user_id":"sv-test","tokens_in":461,"tokens_out":45,"cost_usd":9.615e-05}`. Từ các trường có cấu trúc, tôi có thể (1) nhóm theo `user_id` và cộng `cost_usd` để theo dõi chi phí từng người dùng, và (2) lọc/đếm sự kiện `ask_completed` theo `timestamp` để theo dõi lưu lượng hoặc tìm thời điểm có lỗi. Dòng `print` chỉ có thông báo chữ nên không cung cấp các trường để máy tự động tổng hợp như vậy.

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
| 1 stage (bản đầu) | khoảng 1.100 MB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần chênh lệch chủ yếu đến từ base image Python đầy đủ của bản một stage, các công cụ biên dịch và tệp trung gian chỉ cần khi cài dependency. Bản multi-stage dùng `python:3.11-slim`, cài thư viện ở stage `builder` rồi chỉ chép kết quả cần chạy sang stage `runtime`; `--no-cache-dir` cũng không giữ cache của `pip` trong image cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi tôi chỉ sửa `app/main.py`, các layer tạo base image, đặt `WORKDIR`, `COPY requirements.txt` và `RUN pip install` vẫn được lấy từ cache vì `requirements.txt` không đổi. Từ layer `COPY app ./app` trở đi, Docker phải tạo lại `COPY app`, `COPY utils`, `RUN useradd` và các metadata phía sau. Nếu đặt `COPY . .` trước `RUN pip install`, một thay đổi bất kỳ trong source sẽ làm layer `COPY` đổi, khiến layer cài dependency phía sau mất cache và chạy lại dù `requirements.txt` vẫn y nguyên.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi rủi ro là: lỗ hổng Python cho phép thực thi lệnh từ xa → kẻ tấn công chiếm process của ứng dụng → vì process chạy bằng root nên họ có root trong container → họ tiếp tục lợi dụng capability được cấp, volume nhạy cảm bị mount hoặc lỗ hổng kernel/container runtime để tác động tới host. `USER appuser` cắt giảm đặc quyền ở bước chiếm process: mã độc chỉ nhận quyền của user UID 10001, nên khó sửa file hệ thống hay dùng các thao tác đặc quyền trong container. Biện pháp này giảm mạnh tác động nhưng không thay thế việc giới hạn capability và bảo vệ host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa 20 request: gửi 10 request trong giây cuối của phút trước (ví dụ 10:00:59), rồi gửi tiếp 10 request trong giây đầu của phút sau (10:01:00–10:01:01). Bộ đếm theo phút đã reset ở giây 00 nên chấp nhận cả hai đợt, dù tổng cộng 20 request nằm trong một khoảng 2 giây liên tiếp.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn số request trong cửa sổ 60 giây, còn cost guard giới hạn tổng tiền một user đã dùng trong tháng. Ví dụ, một user chỉ gửi một request trong phút nhưng đã dùng hết ngân sách tháng thì rate limit vẫn cho qua bước kiểm tra tần suất, còn cost guard trả 402. Ngược lại, một user còn gần như toàn bộ ngân sách nhưng gửi request rất rẻ lần thứ 11 trong 60 giây sẽ bị rate limit trả 429, trong khi cost guard vẫn còn khả năng cho qua.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis mất kết nối → endpoint gộp trên cả 3 container đều trả 503 → liveness probe đánh dấu cả 3 container unhealthy → orchestrator có thể restart chúng gần như đồng thời → trong lúc đó cụm không còn instance khỏe để phục vụ, dù bản thân các process Python vẫn sống. Nếu Redis vẫn chưa hồi phục sau khi container khởi động lại, probe tiếp tục thất bại và tạo vòng lặp restart. Tách `/health` khỏi `/ready` giúp load balancer tạm ngừng đưa traffic vào instance nhưng không restart một process còn sống chỉ vì dependency bị lỗi.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis dùng chung, tôi quan sát `history_length` tăng nhất quán `0, 2, 4, ...` rồi giữ ở giới hạn 20, dù các request có thể vào những container khác nhau. Nếu mỗi container dùng một dict Python riêng, mỗi instance chỉ thấy lịch sử do chính nó ghi; vì load balancer phân phối request giữa ba instance, số trả về có thể nhảy giữa các chuỗi như `0, 2, 0, 4...`, thậm chí quay về 0 khi request sang instance chưa có dữ liệu hoặc khi container restart.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Khi kiểm tra deployment trên Render, lệnh gọi `/ask` từng trả `422` với thông báo `JSON decode error`. Tôi xem status code và phần `detail` trong response, rồi đối chiếu request gửi từ PowerShell và phát hiện body JSON bị xử lý sai dấu nháy nên server không parse được. Tôi sửa bằng cách tạo body bằng `ConvertTo-Json` và gửi với `Content-Type: application/json` ở UTF-8; sau đó endpoint trả đúng 401 khi thiếu key và 200 khi gửi key hợp lệ.
