# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Đã điền đầy đủ câu trả lời bên dưới.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Trần Nhứt Nam  Mã học viên: 2A202602981

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Ví dụ thực tế là lúc mình deploy lên Render hay server production mà lỡ quên không điền biến `AGENT_API_KEY` vào dashboard.
Nếu để mặc định `"changeme"`, server vẫn chạy ngon lành và báo healthy. Ai quét trúng URL đều có thể dùng `"changeme"` để gọi chùa endpoint `/ask`, làm tốn tiền token OpenAI/LLM mà mình không hề hay biết cho đến khi bị trừ tiền.
Để trống (bắt buộc) thì app sẽ văng lỗi `ValidationError` và dừng ngay lúc vừa khởi động. Render sẽ báo build/deploy fail liền, mình nhìn log là biết ngay do quên key để vào set lại trước khi public ra ngoài.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T09:10:37.123456+00:00", "user_id": "sv-test", "tokens_in": 43, "tokens_out": 47, "cost_usd": 3.465e-05}`

Hai việc làm được mà `print("đã trả lời xong")` chịu thua:
1. Tự động filter và search log trên các công cụ như CloudWatch, Grafana hay tab Log của Render. Mình có thể query chính xác xem user `sv-test` đã gọi những request nào, hoặc lọc riêng những log báo lỗi (`level == "error"`).
2. Tính toán thống kê tự động: Hệ thống có thể bóc trường `cost_usd` để cộng dồn xem hôm nay tốn bao nhiêu tiền, hoặc đếm tổng số `tokens_in` / `tokens_out` của từng người dùng để vẽ biểu đồ giám sát.

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
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng giảm được gần 750 MB chủ yếu là do:
1. Base image: `python:3.11` gốc chứa cả hệ điều hành Debian đầy đủ cùng trình biên dịch GCC, make, thư viện C header... còn `python:3.11-slim` đã bỏ hết các công cụ build này.
2. Cơ chế multi-stage: Khi cài thư viện ở stage `builder`, các file wheel tải về, cache của pip và file rác trung gian sinh ra chỉ nằm ở stage 1. Sang stage `runtime`, mình chỉ bê mỗi thư mục `/install` sang `/usr/local`, không mang theo bất kỳ file rác nào nên image rất nhẹ.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Khi sửa 1 ký tự trong `app/main.py`: Docker dùng lại cache từ đầu cho đến các bước `COPY requirements.txt .`, `RUN pip install` và `COPY --from=builder /install /usr/local` (hiện `CACHED`). Nó chỉ chạy lại từ lệnh `COPY app ./app` trở đi, nên build lại mất chưa tới 2 giây.
- Nếu đặt `COPY . .` trước `RUN pip install`: Mỗi lần sửa code dù chỉ 1 dấu cách, hash của thư mục thay đổi làm Docker bỏ toàn bộ cache phía sau. Khi đó lệnh `RUN pip install` sẽ bị chạy lại từ đầu, bắt tải lại toàn bộ packages từ mạng, vừa lâu vừa tốn băng thông.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện rủi ro: Giả sử code Python có lỗ hổng (như bị injection hoặc lỗi từ thư viện cài thêm) cho phép kẻ xấu chạy lệnh shell. Nếu container chạy bằng root, tiến trình shell của hacker sẽ có quyền root (UID 0). Tiếp đó, nếu hacker tìm được cách thoát khỏi container (như qua lỗ hổng kernel Linux hay volume mount), khi ra ngoài máy host hacker vẫn giữ nguyên quyền UID 0, tức là chiếm trọn quyền root của cả con server thật.
- Lệnh `USER appuser` cắt đứt chuỗi này: Khi tạo và chuyển sang user thường (`uid 10001`), dù app có bị chiếm quyền shell bên trong container thì hacker cũng chỉ là user không quyền hạn, không sửa được file hệ thống và không thể leo thang đặc quyền để thoát ra máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20 request trong 2 giây liên tiếp**.
Cách đạt được: Giả sử giới hạn là 10 req/phút. Người dùng chờ đến giây 10:00:59 gửi liền 10 request. Sang giây 10:01:00, đồng hồ nhảy sang phút mới nên bộ đếm tự reset về 0, người dùng gửi tiếp 10 request nữa trong giây 10:01:00 hoặc 10:01:01. Vậy là trong đúng 2 giây giáp ranh đó, server phải hứng 20 request (gấp đôi hạn mức) mà hệ thống đếm theo phút đồng hồ vẫn tính là hợp lệ. Thuật toán sliding window 60s sẽ cộng dồn chuẩn xác 60s gần nhất nên sẽ chặn đứng được tình huống này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Khác nhau: Rate limit chặn theo **tốc độ / số request trong thời gian ngắn** (ví dụ: tối đa 10 request/phút) để chống nghẽn server. Còn Cost guard chặn theo **ngân sách tiền tích lũy trong tháng** (ví dụ: tối đa $10/tháng) để tránh bị vỡ nợ tiền API LLM.
- Rate limit cho qua nhưng Cost guard chặn: User gọi rất chậm, cả ngày mới gửi 1 câu hỏi, nhưng tài khoản của user đó trong tháng đã lỡ xài hết $10 tiền quota. Rate limit thấy gọi chậm thì cho qua, nhưng Cost guard sẽ chặn lại và báo lỗi `402 Payment Required`.
- Ngược lại: User mới tạo nick còn nguyên $10 trong tài khoản, nhưng mở tool spam liên tục 15 request trong vòng vài giây. Cost guard thấy chưa hết tiền nên duyệt, nhưng Rate limit sẽ chặn từ request thứ 11 và trả về `429 Too Many Requests`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Nếu gộp làm một và cho check Redis, khi Redis chập chờn mất mạng 30s:
1. Endpoint check Redis fail và trả về 503.
2. Orchestrator (Docker/Kubernetes/Render) tưởng app bị treo/chết tiến trình nên lập tức ra lệnh kill và restart container.
3. Cả 3 container bị restart cùng lúc. Khi vừa bật lên lại check Redis vẫn thấy mất mạng -> lại trả 503 -> lại bị kill tiếp.
4. Hậu quả là cả cụm 3 container rơi vào vòng lặp restart liên tục (CrashLoopBackOff), vừa không phục vụ được request nào vừa làm nghẽn CPU server, dù thực tế bản thân app Python không hề có lỗi gì.
Vì vậy `/health` chỉ check xem app còn sống không, còn `/ready` mới check kết nối Redis để điều hướng traffic.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Khi lưu bằng Redis (Stateless): Dù load balancer chia request vào instance nào thì biến `history_length` vẫn tăng đều đặn qua mỗi câu hỏi (0 -> 2 -> 4 -> 6...) vì tất cả container đều dùng chung một chỗ lưu.
- Nếu lưu bằng dict Python trong RAM: Con số `history_length` sẽ nhảy lung tung. Ví dụ câu 1 vào container A thì A lưu độ dài 2; câu 2 load balancer đẩy sang container B, B không có dữ liệu gì nên coi như cuộc hội thoại mới và độ dài lại về 0. Người dùng chat sẽ thấy bot lúc nhớ lúc quên.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- Lỗi gặp phải: Lúc deploy lên Render, ban đầu deploy bị timeout health check vì app không nhận cổng động. Render tự cấp cổng qua biến môi trường `$PORT` (ví dụ cổng 10000), còn uvicorn trong Dockerfile ban đầu fix cứng `--port 8000`.
- Cách tìm nguyên nhân: Mở tab Logs trên dashboard của Render, thấy uvicorn log thông báo `Application startup complete` trên `0.0.0.0:8000`, nhưng router của Render lại liên tục ping cổng `$PORT` khác và báo `Health check failed`.
- Cách sửa: Sửa lệnh CMD trong `Dockerfile` thành dạng shell `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]` để uvicorn tự đọc biến `$PORT` do Render cấp, và điền đúng path `/health` vào cấu hình `render.yaml`. Sau khi deploy lại thì service chuyển sang Live ngay.
