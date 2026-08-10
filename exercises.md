# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời của bạn vào các mục tương ứng bên dưới.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: NGUYỄN ĐỨC THÀNH  Mã học viên: 2A202601872

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu để mặc định là "changeme", app vẫn khởi động bình thường trên Production. Khi hacker vô tình mò ra token "changeme", họ có thể dùng cạn kiệt tài nguyên (vượt Token Bucket/Cost Guard). Việc "chết sớm" (Fail fast) giúp ta phát hiện ngay lỗi cấu hình thiếu biến môi trường ngay ở khâu deploy (Deploy sẽ fail và báo đỏ), thay vì để hệ thống chạy ngầm với một lỗ hổng bảo mật chết người.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log: `{"event": "chat_completed", "client_id": "EM-KO5NO...", "prompt_tokens": 12, "completion_tokens": 15, "usd_cost": 0.0005, "timestamp": "2026-08-11T12:00:00Z"}`
> Hai việc làm được: 1. Có thể dùng các hệ thống ELK/Datadog tự động đọc, trích xuất field "usd_cost" để vẽ biểu đồ tổng chi phí theo ngày. 2. Có thể tìm kiếm/filter chính xác tất cả các log của một "client_id" cụ thể để truy vết lỗi khi có phàn nàn từ khách hàng đó (print text thô máy không thể bóc tách dữ liệu được).

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | ~1.0 GB |
| Multi-stage | ~150 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch khổng lồ (hơn 800MB) chính là các công cụ build (gcc, make), mã nguồn thư viện C++ (headers), các file cache tải về của pip (pip cache) và các môi trường không cần thiết cho quá trình chạy runtime. Multi-stage build đã vứt bỏ toàn bộ "rác" này đi, chỉ copy lại các file chạy thực tế sang một base image siêu nhẹ (python:3.11-slim) ở Stage 2.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Nếu sửa `app/main.py`, các layer trước đó (như `COPY requirements.txt` và `RUN pip install`) KHÔNG bị thay đổi nên Docker dùng lại từ cache ngay lập tức. Chỉ các layer từ lệnh `COPY . .` trở xuống mới phải chạy lại. 
> Nếu đặt `COPY . .` lên trước `RUN pip install`, thì mỗi lần sửa 1 dòng code nhỏ, Docker sẽ thấy file thay đổi ──► đánh vỡ cache từ đó trở đi ──► buộc phải chạy lại lệnh `pip install` tải lại toàn bộ thư viện cực kỳ tốn thời gian.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Lỗ hổng trong code (vd: cho phép đọc file hoặc thực thi lệnh tùy ý) ──► Kẻ tấn công chạy lệnh độc hại qua app ──► Vì app chạy bằng quyền root, lệnh đó được thực thi bằng root bên trong container ──► Kẻ tấn công có thể lợi dụng lỗi thoát container (container breakout/privilege escalation) để chiếm quyền root trên máy chủ (host).
> Lệnh `USER appuser` cắt đứt chuỗi này vì app chỉ chạy với quyền user không đặc quyền, nên dù chiếm được app, kẻ tấn công cũng không thể cài đặt mã độc hay can thiệp hệ thống.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> Phải kèm `WWW-Authenticate: Bearer` để tuân thủ chuẩn HTTP/1.1 (RFC 6750), báo cho phần mềm client/trình duyệt biết server đang yêu cầu phương thức xác thực nào để tự động prompt. 
> Trả cùng một thông báo lỗi là để chống lại tấn công dò đoán (Enumeration Attack/Timing Attack). Nếu báo rõ "sai token", kẻ tấn công sẽ biết format của họ đúng nhưng token sai, dễ bề thu hẹp phạm vi để brute-force.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Một client im lặng 10 phút sẽ chỉ gửi được tối đa 10 request trước khi bị lỗi 429. Bởi vì bucket có sức chứa tối đa là 10 (`capacity=10`). 
> Nếu bỏ đoạn `min(capacity, ...)`, số token sẽ được cộng dồn mãi mãi không có trần (sau 10 phút x 10 = 100 token). Khi đó client có thể xả spam hàng trăm request cùng một lúc, gây quá tải hệ thống, phá vỡ nguyên lý giới hạn sốc (Burst Limit) của Token Bucket.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Hạn mức $30/tháng: Thiệt hại tối đa là mất trắng $30 ngay trong đêm đó, client đó bị chặn vĩnh viễn đến hết tháng (chết 29 ngày còn lại).
> Hạn mức $1/ngày: Thiệt hại tối đa chỉ là $1 đêm đó. Service tự động hồi phục và cấp ngân sách mới cho client đó vào 0h ngày hôm sau, chỉ gián đoạn phần còn lại của ngày hôm đó. Quản lý rủi ro và trải nghiệm người dùng tốt hơn nhiều.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp chung, khi Redis sập 30 giây: /healthz sẽ trả về 503 (báo chết) ──► Nền tảng Cloud/Kubernetes nghĩ 3 container đã hỏng ──► Gửi tín hiệu SIGTERM để kill và khởi động lại toàn bộ 3 container cùng lúc ──► Gây downtime sập toàn hệ thống (Cascading failure). 
> Nếu tách riêng: /readyz báo đỏ (Cloud ngưng chia tải request mới vào) nhưng /healthz báo xanh (container vẫn sống). Khi Redis hồi phục sau 30s, /readyz tự xanh lại, hệ thống tự phục hồi mà không hề bị restart cái nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Thông báo lỗi: Gặp lỗi 502 Bad Gateway khi truy cập URL và lệnh `python grade.py` rớt các bài test CP5 do không kết nối được Redis thật.
> Nguyên nhân: Do trong quá trình setup Railway, biến môi trường `REDIS_URL` của con Web bị trỏ nhầm về chính cái internal URL của nó (tự gọi vào cổng 6379 của mình).
> Cách sửa: Vào mục Variables của service Web, xóa biến REDIS_URL cũ đi, bấm "Add Reference Variable" và trích xuất tham chiếu trực tiếp đến `REDIS_URL` của cục Redis thứ hai vừa tạo. Chờ hệ thống deploy lại báo Active là thành công.
