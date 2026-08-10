# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay các dòng trả lời mẫu bên dưới bằng câu của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: ..........................  Mã học viên: ..........................

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy lên cloud mà quên set `AGENT_API_KEY`, app sẽ chết ngay lúc khởi động
thay vì lặng lẽ chạy với khóa mặc định. Như vậy mình phát hiện lỗi cấu hình
trước khi service công khai đi vào production. Nếu để mặc định kiểu
`changeme`, app vẫn sống nhưng ai cũng có thể gọi `/ask` bằng khóa đó, rồi
mình chỉ biết chuyện khi log và hóa đơn LLM phình ra.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Một dòng log thật từ `/ask` có dạng:

```json
{"event":"ask_completed","level":"info","timestamp":"2026-08-10T05:25:53+00:00","user_id":"sv-test","tokens_in":12,"tokens_out":34,"cost_usd":0.0004}
```

Từ dòng này mình có thể lọc theo `event`/`level` để thống kê số lần hỏi hoặc
debug lỗi trong production, và có thể gom theo `user_id` để tìm user nào đang
gây tốn chi phí. `print("đã trả lời xong")` không cho mình machine-readable
fields nên không query, đếm, hay cảnh báo tự động được.

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
| 1 stage (bản đầu) | 1.73GB |
| Multi-stage | 270MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Bản 1 stage là `1.73GB`, còn bản multi-stage là `270MB`. Phần chênh lệch chủ
yếu là toàn bộ base image nặng và lớp build không cần ở runtime: compiler,
wheel cache, metadata cài đặt tạm thời, và các thứ chỉ dùng để build dependency.
Multi-stage chỉ giữ lại những gì app thật sự cần để chạy.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Với Dockerfile hiện tại, sửa một ký tự trong `app/main.py` chỉ làm layer `COPY
app ./app` chạy lại; layer `COPY requirements.txt` và `RUN pip install` vẫn lấy
từ cache vì file requirements không đổi. Nếu đặt `COPY . .` trước `RUN pip
install`, chỉ cần sửa một dòng code là toàn bộ cache phía sau bị phá, nên Docker
phải cài lại dependency từ đầu mỗi lần build.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Nếu container chạy bằng root, một lỗ hổng RCE trong Python có thể cho kẻ tấn
công sửa file hệ thống trong container, cài thêm công cụ, đọc dữ liệu nhạy cảm
trong volume gắn vào, và khi có cơ hội escape khỏi container thì quyền root sẽ
nguy hiểm hơn nhiều. Lệnh `USER` cắt đứt chuỗi đó bằng cách cho app chạy dưới
user thường, nên ngay cả khi bị exploit thì mức phá hoại vẫn bị giới hạn.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Nếu đếm theo phút đồng hồ, một user có thể gửi tối đa `20` request trong 2 giây
liên tiếp: 10 request ở cuối một phút, rồi ngay khi đồng hồ sang phút mới lại
gửi thêm 10 request nữa. Sliding window 60 giây chặn đúng kiểu bùng nổ này vì
nó luôn nhìn 60 giây gần nhất thay vì reset theo mốc phút.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit chặn theo số lượng request trong một khoảng thời gian, còn cost guard
chặn theo số tiền đã tiêu. Ví dụ, 1 user có thể gửi rất nhiều request rẻ tiền
đến mức rate limit cho qua nhưng tổng tiền vẫn thấp; ngược lại, vài request rất
đắt có thể làm cost guard chặn dù rate limit vẫn chưa chạm ngưỡng. Với lab này,
rate limit bảo vệ tần suất gọi API, cost guard bảo vệ ngân sách.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Nếu gộp `/health` và `/ready` thành một endpoint rồi bắt nó kiểm tra Redis, khi
Redis mất 30 giây thì cả 3 container sẽ bị báo unhealthy cùng lúc. Load balancer
sẽ thấy mọi instance đều “chết”, đẩy traffic lung tung hoặc restart liên tục,
trong khi bản thân process vẫn còn sống. Tách riêng `/health` và `/ready` giúp
`/health` chỉ báo process còn sống, còn `/ready` mới báo có sẵn sàng nhận traffic
hay không.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lịch sử nằm trong một dict Python thay vì Redis, `history_length` sẽ phụ
thuộc container nào nhận request. Cùng một `X-User-Id`, mình có thể thấy 0 ở
instance A, rồi 2 ở instance B nhưng sau đó lại quay về 0 nếu request tiếp theo
đi vào instance C. Nói cách khác, số này sẽ không tăng ổn định trên toàn cụm và
state sẽ bị chia cắt theo từng process.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Khi deploy lên Railway, mình gặp lỗi health check fail sau khi build/deploy đã
pass. Tìm nguyên nhân bằng cách mở log và thử chạy local thì thấy app vẫn chạy
được trên cổng 8000, nhưng cloud cần app đọc biến `PORT`. Mình sửa Dockerfile
để `uvicorn` và healthcheck dùng `${PORT:-8000}`, rồi redeploy và service lên
được bình thường.
