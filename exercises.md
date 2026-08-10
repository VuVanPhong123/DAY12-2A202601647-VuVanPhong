# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Các câu trả lời dưới đây dựa trên code và những lần
> chạy thực tế trong worktree này.

Họ và tên: Vũ Văn Phong
Mã học viên: 2A202601647

---

### Câu 1 — Fail fast (CP1)

Nếu deploy lên cloud mà quên đặt `AGENT_API_KEY`, `Settings` sẽ báo lỗi ngay
trong lúc khởi động. Nhờ vậy mình biết cấu hình thiếu trước khi mở public URL.
Nếu mặc định là `changeme`, service vẫn chạy và người khác có thể đoán được
khóa để gọi API, đến khi thấy hóa đơn mới phát hiện ra.

### Câu 2 — Log cho máy đọc (CP1)

Một dòng log thật lấy từ container sau khi gọi `/ask` local:

```text
{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T03:03:04.520089+00:00", "user_id": "smoke-user", "tokens_in": 4, "tokens_out": 36, "cost_usd": 2.22e-05}
```

Từ dòng này có thể lọc theo `user_id` để đếm hoạt động của từng user và cộng
`cost_usd` để theo dõi chi phí. Cũng có thể tổng hợp `tokens_in` và
`tokens_out` theo thời gian; một câu `print("đã trả lời xong")` không có các
trường dữ liệu ổn định để truy vấn tự động.

### Câu 3 — Kích thước image (CP2)

Mình build và đo bằng `docker images`:

| Bản | Dung lượng |
|-----|-----------:|
| 1 stage (bản starter tương ứng) | 1.73GB |
| Multi-stage | 270MB |

Bản 1 stage dùng base `python:3.11` đầy đủ, giữ cả phần môi trường lớn hơn và
copy nhiều file của repo. Bản multi-stage dùng `python:3.11-slim`, chỉ copy
dependency đã cài cùng `app` và `utils` sang runtime nên không mang theo phần
build/context không cần thiết.

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Mình thêm một comment vô hại vào `app/main.py`, build lại rồi hoàn nguyên. Log
build cho thấy `COPY requirements.txt`, `pip install` và
`COPY --from=builder /install` đều `CACHED`. Layer `COPY app ./app` phải build
lại; các layer phía sau nó (`COPY utils`, tạo user và export image) cũng chạy
lại do Docker cache phụ thuộc vào thứ tự layer. Nếu `COPY . .` đặt trước
`pip install`, chỉ một thay đổi trong source cũng làm layer copy đổi và Docker
phải cài lại toàn bộ dependency.

### Câu 5 — Vì sao không chạy bằng root (CP2)

Nếu code có lỗ hổng dẫn tới thực thi lệnh, tiến trình trong container có thể
bị lợi dụng để đọc/sửa file hoặc chạy process với quyền root. Nếu container
root bị thoát ra ngoài do lỗi cấu hình/runtime, quyền đó có thể mở rộng thành
quyền rất cao trên host. `USER appuser` làm tiến trình ứng dụng chạy bằng user
không đặc quyền, nên cắt chuỗi tấn công ở bước quyền trong container.

### Câu 6 — Cửa sổ trượt (CP3)

Tối đa là 20 request trong 2 giây. Người dùng gửi 10 request ở giây
10:00:59, hệ thống theo phút đồng hồ vẫn tính chúng cho phút 10:00; sau đó
gửi tiếp 10 request ở 10:01:01, hệ thống lại tính cho phút 10:01. Sliding
window nhìn vào 60 giây gần nhất nên không có khe hở này và sẽ chặn nhóm thứ
hai nếu nhóm thứ nhất vẫn còn trong cửa sổ.

### Câu 7 — Rate limit và cost guard (CP3)

Rate limit giới hạn số lần gọi trong một cửa sổ thời gian, còn cost guard
giới hạn tổng tiền của từng user trong từng tháng. Ví dụ user còn hạn mức
request nhưng đã gần hết ngân sách tháng, cost guard phải chặn khi chi phí dự
kiến làm vượt ngân sách. Ngược lại, user có thể còn rất nhiều ngân sách nhưng
gửi quá 10 request trong 60 giây; rate limiter phải trả 429.

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint và để cả hai kiểm tra Redis, khi Redis mất kết nối cả ba
container sẽ cùng báo unhealthy/503. Orchestrator có thể restart cả ba cùng
lúc thay vì chỉ ngừng nhận traffic ở những instance chưa ready. Redis quay lại
nhưng cụm vừa bị restart đồng loạt có thể không còn instance phục vụ, biến
một sự cố dependency ngắn thành sự cố toàn hệ thống. Tách `/health` (liveness,
không chạm Redis) khỏi `/ready` (readiness, có chạm Redis) tránh việc đó.

### Câu 9 — Stateless (CP4)

Mình chạy 3 agent sau Nginx và gọi cùng `X-User-Id: scale-user` năm lần. Kết
quả thật của `history_length` là `0, 2, 4, 6, 8`. Các request có thể đi vào
container khác nhau nhưng Redis dùng chung nên lịch sử vẫn tăng đều. Nếu dùng
dict Python riêng trong mỗi process, giá trị sẽ phụ thuộc container nhận
request, có thể quay về 0 hoặc tăng theo từng container thay vì thể hiện một
lịch sử chung.

### Câu 10 — Deploy thật (CP5)

`PENDING MANUAL EVIDENCE`: lượt này chưa thực hiện deploy cloud nên mình không
bịa lỗi, log hay cách sửa. Sau khi deploy thủ công, cần cập nhật một lỗi hoặc
quan sát thật từ dashboard/log (nếu có), nguyên nhân đã xác minh và cách sửa
vào câu này.
