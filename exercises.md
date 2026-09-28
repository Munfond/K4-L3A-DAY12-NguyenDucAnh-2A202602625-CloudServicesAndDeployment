# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời bên dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Đức Anh  Mã học viên: 2A202602625

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy ứng dụng lên cloud hoặc môi trường production, nếu lập trình viên quên cấu hình biến môi trường `AGENT_API_KEY` trên Dashboard:
- Nếu để giá trị mặc định `"changeme"`: Ứng dụng vẫn khởi động thành công và container báo trạng thái healthy. Tuy nhiên, khóa mặc định này nằm công khai trong mã nguồn, kẻ xấu có thể quét và sử dụng ngay khóa `"changeme"` để gọi API miễn phí, đốt sạch ngân sách và hạn mức LLM trước khi quản trị viên kịp phát hiện.
- Khi không có giá trị mặc định (Fail Fast): Ứng dụng sẽ crash ngay lập tức lúc khởi động. Nền tảng (Render/Docker) sẽ phát hiện tiến trình thoát với mã lỗi và đánh dấu deploy thất bại, ngăn chặn service nhận bất kỳ traffic nào từ bên ngoài và cảnh báo lập trình viên bổ sung secret kịp thời.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
`{"timestamp": "2026-09-28T11:09:55.123456Z", "level": "info", "event": "ask_completed", "user_id": "sv-test", "tokens_in": 48, "tokens_out": 52, "cost_usd": 0.0000384}`

Hai việc làm được với log có cấu trúc (JSON structured logging):
1. **Lọc, tổng hợp và phân tích dữ liệu tự động bằng các hệ thống tập trung (Elasticsearch/Datadog/CloudWatch):** Có thể viết truy vấn tính tổng số token hoặc tổng chi phí theo từng `user_id` trong tháng (`SELECT sum(cost_usd) GROUP BY user_id`). Với `print()`, log là dạng text phi cấu trúc, cực kỳ khó và tốn kém khi parse.
2. **Thiết lập cảnh báo tự động theo ngưỡng (Alerting):** Có thể cấu hình hệ thống giám sát tự động bắn cảnh báo khi `cost_usd` của một request vượt ngưỡng bất thường hoặc khi trường `level` chuyển thành `error`, giúp đội vận hành phản ứng tức thì.

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
| 1 stage (bản đầu) | ~460 MB |
| Multi-stage | ~185 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~275 MB) bao gồm: bộ công cụ quản lý package `pip`, thư mục cache tải về của pip (`~/.cache/pip`), các trình biên dịch/header C tạm thời sinh ra trong quá trình cài đặt wheel, và các file rác phát sinh khi apt-get cài đặt dependencies. Multi-stage build loại bỏ toàn bộ những thứ này bằng cách chỉ sao chép thư mục `/opt/venv` và mã nguồn sang image runtime tối giản.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại: Các layer từ base image, tạo virtualenv, `COPY requirements.txt .`, và `RUN pip install ...` đều được tái sử dụng từ Docker cache (CACHED) vì `requirements.txt` không thay đổi. Docker chỉ phải chạy lại từ bước `COPY . .` trở về sau. Thời gian build lại chỉ mất 1-2 giây.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Bất cứ khi nào sửa dù chỉ một ký tự trong `app/main.py`, cache của layer `COPY . .` sẽ bị hủy (cache bust), kéo theo toàn bộ lệnh `RUN pip install` phía sau buộc phải chạy lại từ đầu, khiến thời gian build kéo dài hàng phút và tốn băng thông mạng.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện tấn công:
1. Kẻ tấn công khai thác lỗ hổng thực thi mã từ xa (RCE) trong code Python (ví dụ qua thư viện deserialization hoặc command injection).
2. Kẻ tấn công chiếm được quyền điều khiển tiến trình trong container. Nếu container chạy với người dùng `root` mặc định (UID 0), kẻ tấn công sở hữu quyền root trong container.
3. Kẻ tấn công lợi dụng các lỗ hổng nhân Linux (kernel exploits) hoặc các volume gắn kết nhạy cảm (như docker socket `/var/run/docker.sock`) để thoát khỏi container (container escape) và chiếm quyền root trực tiếp trên máy host.

Lệnh `USER appuser` (UID 10001) cắt đứt chuỗi tấn công ngay từ bước 2: Khi kẻ tấn công chiếm được process, họ chỉ có quyền của user thường không đặc quyền. Họ không thể sửa file hệ thống, không thể chèn rootkit, và không có quyền thực hiện các thao tác nhạy cảm để leo thang đặc quyền hay container escape.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- Số request tối đa: **20 request** trong 2 giây liên tiếp.
- Cách đạt được:
  Với cách đếm theo phút đồng hồ (fixed window):
  - Người dùng gửi 10 request vào giây cuối cùng của phút thứ nhất: lúc `10:00:59`. Cả 10 request đều hợp lệ vì hạn mức trong phút `10:00` là 10.
  - Sang giây `10:01:00`, bộ đếm được reset về 0 cho phút mới.
  - Người dùng gửi tiếp ngay 10 request nữa lúc `10:01:00` (hoặc `10:01:01`). Cả 10 request này tiếp tục được chấp nhận.
  Như vậy, chỉ trong 2 giây liên tiếp (từ `10:00:59` đến `10:01:01`), người dùng đã gửi thành công 20 request, gây ra đột biến lưu lượng (traffic burst) gấp đôi mức cho phép. Sliding window 60 giây giải quyết triệt để vấn đề này vì luôn tính lùi đúng 60 giây từ thời điểm request đến.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Điểm khác nhau: Rate limit giới hạn **tần suất/số lượng request** trong một đơn vị thời gian ngắn (ví dụ 10 req/phút) để bảo vệ hạ tầng máy chủ khỏi quá tải. Cost guard giới hạn **tổng chi phí tài chính (USD/token)** tích lũy trong tháng để tránh hóa đơn LLM tăng vọt.
- Tình huống Rate limit cho qua nhưng Cost guard chặn: Người dùng chỉ gửi 1 request duy nhất trong ngày (hoàn toàn không chạm ngưỡng 10 req/phút của rate limit), nhưng ngân sách tháng của user này đã cạn kiệt (đạt $10.0). Cost guard sẽ phát hiện và chặn lại với mã 402 Payment Required.
- Tình huống Cost guard cho qua nhưng Rate limit chặn: Người dùng mới bắt đầu tháng, ngân sách còn nguyên $10.0, nhưng gửi liên tiếp 15 request chỉ trong 3 giây (mỗi request câu hỏi ngắn, tốn rất ít token). Cost guard thấy tiền còn nhiều nên cho qua, nhưng Rate limit sẽ chặn từ request thứ 11 với mã 429 Too Many Requests để bảo vệ server.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện xảy ra:
1. Redis gặp sự cố mạng hoặc khởi động lại trong 30 giây.
2. Endpoint gộp kiểm tra thấy Redis không phản hồi nên trả về lỗi HTTP 503.
3. Vì endpoint này kiêm nhiệm vai trò Liveness probe, bộ điều phối (Docker/Kubernetes/Render) cho rằng cả 3 container agent đều đã chết và đồng loạt ra lệnh restart (kill và khởi động lại) cả 3 container.
4. Khi cả 3 container mới khởi động lại, Redis vẫn chưa online trong khoảng 30s đó, nên các container mới lại tiếp tục trả về 503 và tiếp tục bị restart liên tục (hiện tượng CrashLoopBackOff / restart storm).
5. Mọi request của người dùng đang xử lý bị ngắt quãng, tải CPU tăng vọt do khởi động lại liên tục và toàn bộ service bị sập hoàn toàn thay vì chỉ tạm dừng nhận request mới chờ Redis hồi phục.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Khi dùng Redis (Stateless): Dù load balancer phân phối các request luân phiên qua lại giữa container 1, container 2 và container 3, cả 3 container đều truy xuất chung vào cơ sở dữ liệu Redis, do đó `history_length` tăng đều đặn và liên tục: 0 -> 2 -> 4 -> 6...
- Nếu lưu trong dict Python trong bộ nhớ RAM: Mỗi container sở hữu một bộ nhớ riêng biệt. Lượt hỏi 1 rơi vào container 1 (dict của container 1 lưu 2 message). Lượt hỏi 2 rơi vào container 2 (dict container 2 đang rỗng nên thấy `history_length` = 0). Lượt hỏi 3 rơi vào container 3 (cũng rỗng nên `history_length` = 0). Lượt hỏi 4 quay lại container 1 thì mới thấy `history_length` = 2. Giá trị `history_length` sẽ nhảy thất thường và agent bị hiện tượng "mất trí nhớ".

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- Lỗi gặp phải: Quá trình đồng bộ Render Blueprint hiển thị trạng thái "Syncing" kéo dài, và khi truy cập vào link gốc của service thì nhận thông báo `{"detail":"Not Found"}`.
- Cách tìm nguyên nhân:
  1. Với lỗi "Syncing": Mở chi tiết Blueprint trên Render Dashboard thì nhận thấy Render đang tạm dừng để chờ nhập giá trị cho `AGENT_API_KEY` do thiết lập `sync: false` trong `render.yaml`.
  2. Với lỗi `{"detail":"Not Found"}`: Kiểm tra mã nguồn trong `app/main.py` và nhật ký log của service `day12-agent`, phát hiện ứng dụng là một API backend chỉ định nghĩa các router `/health`, `/ready` và `/ask` chứ không có trang chủ tại route `/`.
- Cách sửa:
  1. Nhập chuỗi secret cho `AGENT_API_KEY` trên Dashboard và nhấn "Apply" để Render hoàn tất việc build và chạy cả 2 service `day12-agent` và `day12-redis`.
  2. Truy cập đúng các endpoint `/health`, `/ready` hoặc `/docs` (Swagger UI) để kiểm tra, tất cả đều trả về mã HTTP 200 OK.

