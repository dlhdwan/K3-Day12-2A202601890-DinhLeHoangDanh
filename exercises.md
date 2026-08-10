# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Họ và tên: Đinh Lê Hoàng Danh  Mã học viên: 2A202601890

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống: Khi deploy ứng dụng lên môi trường Production/Staging mà bạn quên chưa cài đặt biến môi trường `AGENT_API_KEY` trên Dashboard.
- Nếu để mặc định `"changeme"`: Ứng dụng vẫn khởi động bình thường. Kẻ tấn công hoặc các bot quét tự động trên Internet có thể dùng khóa mặc định `"changeme"` này để gọi API miễn phí. Bạn sẽ chỉ phát hiện ra khi hóa đơn LLM phình to hoặc hệ thống bị phá hoại.
- Khi áp dụng Fail-Fast (không có mặc định): Đợt deploy sẽ thất bại ngay ở bước khởi động (ngay khi bạn đang nhìn màn hình deployment). Ứng dụng lập tức ngừng hoạt động trước khi có bất kỳ request nào lọt vào, giúp bảo vệ tài khoản và tài chính của bạn ngay từ đầu.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T02:50:18.190786+00:00", "user_id": "sv01", "tokens_in": 39, "tokens_out": 45, "cost_usd": 0.00003285}
```

Hai việc làm được với log JSON có cấu trúc:
1. **Lọc và truy vấn chính xác bằng máy (CloudWatch/Datadog/Grafana Loki)**: Có thể dễ dàng lọc theo các trường dữ liệu cụ thể như `user_id == "sv01"` hoặc tìm các request có `cost_usd > 0.001` trong vòng vài giây mà không phải viết regex phức tạp.
2. **Tự động hóa cảnh báo và vẽ biểu đồ dashboard**: Hệ thống giám sát có thể tự động parse các trường số `cost_usd` hay `tokens_out` để tính tổng chi phí theo giờ/ngày hoặc kích hoạt alert tự động khi tổng chi phí tăng bất thường.

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
| 1 stage (bản đầu) | ~1.02 GB |
| Multi-stage | ~270 MB |

Giải thích: Phần dung lượng chênh lệch (~750 MB) chính là các công cụ biên dịch (gcc, g++, build-essential), cache của pip, và bộ SDK/thư viện phát triển không cần thiết ở môi trường runtime. Multi-stage build đã loại bỏ toàn bộ các công cụ build này ở stage `builder` và chỉ copy các thư viện Python đã cài đặt sang stage runtime gọn nhẹ.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại (`COPY requirements.txt` $\rightarrow$ `RUN pip install` $\rightarrow$ `COPY app/ app/`):
  - Layer `COPY requirements.txt` và `RUN pip install` được dùng lại từ **Cache** hoàn toàn (vì `requirements.txt` không đổi).
  - Chỉ duy nhất layer `COPY app/ app/` (và các layer bên dưới nó) phải chạy lại. Quá trình build chỉ mất chưa đầy 1 giây.
- Nếu đặt `COPY . .` lên trước `RUN pip install`:
  - Khi sửa một ký tự trong `app/main.py`, layer `COPY . .` bị biến đổi checksum $\rightarrow$ toàn bộ Cache từ layer này trở đi bị hủy bỏ.
  - Kết quả: Docker buộc phải chạy lại lệnh `RUN pip install` và tải/cài đặt lại toàn bộ thư viện từ internet, tốn vài phút cho mỗi lần sửa code.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:
1. Code Python gặp lỗ hổng (ví dụ: Remote Code Execution - RCE qua `eval()` hoặc thư viện có lỗ hổng).
2. Kẻ tấn công thực thi lệnh shell tùy ý bên trong container.
3. Nếu container đang chạy dưới quyền `root`, tiến trình tấn công có quyền root trong container. Kết hợp với một lỗ hổng container breakout (hoặc mount socket Docker), kẻ tấn công chiếm luôn quyền `root` trên hệ điều hành máy host.
4. Lệnh `USER appuser` cắt đứt chuỗi này ngay ở **bước 3**: Khi chuyển sang user thường (`appuser`), kẻ tấn công bị giới hạn quyền nghiêm ngặt bên trong container, không thể đọc/ghi các file hệ thống quan trọng hay thực hiện các thao tác quản trị cao cấp.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.
Giải thích: Người dùng gửi 10 request ở giây `10:00:59` (phút thứ 00) và gửi tiếp 10 request ở giây `10:01:01` (phút thứ 01). Nếu đếm theo phút đồng hồ, cả 2 đợt gửi này đều nằm trong hạn mức 10 req/phút của từng phút riêng biệt, nhưng tính theo cửa sổ trượt thời gian thực thì người dùng đã spam 20 request chỉ trong 2 giây.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Khác nhau:
- **Rate Limit**: Giới hạn tần suất/số lượng request trong một đơn vị thời gian (req/phút) để bảo vệ hạ tầng khỏi bị quá tải/spam.
- **Cost Guard**: Giới hạn tổng chi phí/ngân sách (USD) trong tháng để bảo vệ tài chính khỏi các request tiêu tốn quá nhiều token.

Tình huống:
1. **Rate Limit CHO QUA nhưng Cost Guard CHẶN**: User chỉ gửi 2 request/phút (đạt Rate Limit), nhưng mỗi request gửi một file PDF dài 500 trang ($6.00 USD/request). Ngân sách tháng là $10.00 USD. Request 1 hết $6.00, sang Request 2 dự kiến $12.00 $\rightarrow$ Cost Guard chặn (trả 402 Payment Required) mặc dù Rate Limit vẫn hợp lệ.
2. **Cost Guard CHO QUA nhưng Rate Limit CHẶN**: User mới dùng $0.05 USD trong ngân sách $10.00 USD. User chạy script gửi 15 request cực ngắn ("hi", "hello") trong 3 giây. Ngân sách còn nhiều nhưng người dùng gửi 15 req/3s $\rightarrow$ Rate Limit chặn ở request thứ 11 (trả 429 Too Many Requests).

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện:
1. Kết nối Redis chập chờn hoặc rớt trong 30 giây.
2. Endpoint gộp báo lỗi và trả về HTTP `503 Unhealthy`.
3. Orchestrator (Docker/Kubernetes/Cloud) thấy endpoint liveness báo unhealthy $\rightarrow$ coi tiến trình app đã bị treo/hỏng và tiến hành **kiết (kill) và restart lại toàn bộ 3 container**.
4. Cả 3 container bị restart cùng lúc. Khi Redis vừa có lại kết nối, hệ thống không còn container nào sẵn sàng phục vụ.
5. Sự cố chập chờn Redis nhỏ biến thành sự cố toàn bộ hệ thống bị sập (Downtime toàn phần).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lịch sử lưu trong dict Python (Stateful trong RAM từng container):
Con số `history_length` sẽ **thay đổi thất thường và không tăng đều** (ví dụ: request 1 rơi vào container A trả về 0, request 2 rơi vào container B trả về 0, request 3 rơi vào A trả về 2, request 4 rơi vào container C trả về 0...). Agent sẽ bị "mất trí nhớ" ngẫu nhiên tùy thuộc vào container nào tiếp nhận request.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi gặp phải: `Error: Invalid value for '--port': '$PORT' is not a valid integer` dẫn tới `Healthcheck failure` trên Railway.
- **Thông báo lỗi**: `Error: Invalid value for '--port': '$PORT' is not a valid integer.` trong tab Deploy Logs.
- **Nguyên nhân**: File `railway.toml` cấu hình `startCommand = "uvicorn app.main:app --host 0.0.0.0 --port $PORT"`. Do Railway thực thi lệnh trực tiếp không qua shell, chuỗi `$PORT` không được nội suy thành số cổng thực tế (ví dụ: 7011) mà truyền thẳng chuỗi `"$PORT"` làm uvicorn crash lúc khởi động.
- **Cách sửa**: Sửa lại `startCommand` trong `railway.toml` thành `sh -c 'exec uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}'` để chạy qua shell wrapper và nội suy biến `$PORT` thành công.
