# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lê Khả Chính  Mã học viên: 2A202601852

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên cloud, nếu quên đặt `API_TOKEN` thì ứng dụng dừng ngay ở bước
> khởi động và log báo thiếu cấu hình. Nhờ vậy bản deploy lỗi không bao giờ nhận
> traffic. Nếu dùng mặc định `"changeme"`, service vẫn báo healthy nhưng bất kỳ
> ai biết hoặc đoán được token mẫu đều có thể gọi `/chat` và tiêu ngân sách LLM.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một log thực tế có dạng:
> `{"event":"chat_completed","severity":"INFO","ts":"2026-08-10T10:24:31+00:00","client_id":"cp5-verify","prompt_tokens":3,"completion_tokens":43,"usd_cost":0.00002625}`.
> Với JSON này, hệ thống log có thể (1) lọc/đếm số lần `chat_completed` theo
> `client_id` và (2) cộng `usd_cost`, lập biểu đồ hoặc cảnh báo khi chi phí vượt
> ngưỡng. Câu `print("đã trả lời xong")` không có các trường có cấu trúc để máy
> thực hiện hai việc đó một cách đáng tin cậy.

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
| 1 stage (bản đầu) | Chưa đo được trên máy hiện tại |
| Multi-stage | Chưa đo được trên máy hiện tại |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Máy làm bài hiện không có Docker nên hai test build thật được skip; vì vậy tôi
> không ghi số MB giả. Về cấu tạo, bản đầu dùng image Python đầy đủ và giữ cả
> công cụ build, cache cùng dependency trung gian. Bản multi-stage dùng
> `python:3.11-slim`; compiler và dữ liệu chỉ cần lúc build nằm lại ở stage
> `builder`, còn stage runtime chỉ nhận virtual environment và mã cần chạy nên
> nhỏ hơn đáng kể.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, các layer base image, `COPY requirements.txt` và
> `RUN pip install` vẫn được dùng lại từ cache; chỉ layer copy source và các
> layer sau nó phải tạo lại. Nếu đặt `COPY . .` trước `RUN pip install`, thay đổi
> một ký tự trong bất kỳ file source nào cũng làm layer copy đổi, khiến Docker
> mất cache và cài lại toàn bộ dependency dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Kẻ tấn công khai thác lỗi Python để thực thi lệnh trong container. Nếu process
> chạy bằng root, lệnh đó cũng có UID 0 trong container; khi có thêm cấu hình
> nguy hiểm như mount Docker socket, mount thư mục host hoặc một lỗ hổng runtime,
> kẻ tấn công có thể sửa dữ liệu hay chiếm quyền cao trên host. `USER appuser`
> cắt chuỗi ngay sau bước thực thi mã: mã độc chỉ có quyền của user thường, bị
> hạn chế truy cập file, thiết bị và các tài nguyên đặc quyền.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> `WWW-Authenticate: Bearer` là tín hiệu chuẩn cho client biết endpoint yêu cầu
> cơ chế Bearer và cho phép thư viện HTTP xử lý response 401 đúng chuẩn. Dùng
> cùng thông báo cho thiếu header, sai scheme và sai token tránh tiết lộ bước
> kiểm tra nào đã đúng. Nếu trả lỗi quá chi tiết, người dò token có thêm thông
> tin để thu hẹp cách tấn công và phân biệt token hợp lệ với request sai định
> dạng.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Client chỉ gửi liên tiếp được 10 request trước khi nhận 429, vì xô không bao
> giờ chứa quá `capacity=10`. Nếu bỏ `min(capacity, ...)`, sau 10 phút nó tích
> thêm `10 × 10 = 100` token; nếu trước lúc chờ xô đang đầy thì tổng có thể thành
> 110 token, nên client có thể bắn 110 request liên tiếp. Đây là burst không bị
> giới hạn và làm mất ý nghĩa của sức chứa xô.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Với hạn mức 30 USD/tháng, sự cố có thể tiêu tối đa toàn bộ 30 USD trước khi bị
> chặn và service chỉ tự dùng lại được khi sang tháng mới (hoặc quản trị viên can
> thiệp). Với hạn mức 1 USD/ngày, thiệt hại tối đa trong ngày xảy ra sự cố là 1
> USD; request bị chặn đến 00:00 UTC, sau đó khóa ngày mới giúp service tự phục
> hồi. Hạn mức ngày thu hẹp đáng kể phạm vi thiệt hại và thời gian gián đoạn.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis mất kết nối → endpoint gộp trả 503 → liveness probe coi cả ba container
> là hỏng → orchestrator restart cả ba gần như cùng lúc → container mới vẫn
> không kết nối được Redis và tiếp tục fail/restart. Một lỗi dependency tạm thời
> vì thế biến thành việc toàn bộ service không còn instance ổn định. Khi tách
> endpoint, `/healthz` vẫn 200 để process không bị restart, còn `/readyz` trả 503
> để load balancer chỉ tạm ngừng gửi traffic cho tới khi Redis phục hồi.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Sau khi Render deploy thành công, test authenticated chat ban đầu trả
> `401 {"detail":"invalid or missing bearer token"}` trong khi `/healthz` và
> `/readyz` đều 200. Tôi so sánh độ dài `API_TOKEN` và `DEPLOY_API_TOKEN` trong
> `.env` và phát hiện token kiểm thử thiếu một ký tự, không phải lỗi service.
> Tôi đồng bộ lại đúng token đang đặt trên Render rồi chạy lại CP5; request có
> Bearer token trả 200 và kết quả cuối là 9 test cloud pass.
