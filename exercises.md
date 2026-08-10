# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng câu trả lời của bạn bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Trần Duy Trường  Mã học viên: 2A202601247

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu để mặc định là "changeme", khi triển khai lên Cloud mà quên thiết lập khóa bí mật AGENT_API_KEY trong dashboard, ứng dụng vẫn sẽ khởi động bình thường. Khi đó, các bot quét tự động có thể gọi API bằng API Key mặc định hoàn toàn miễn phí và đốt sạch ngân sách API LLM của bạn mà bạn không hề hay biết cho đến khi nhận được hóa đơn thanh toán. Việc "chết sớm" (fail-fast) giúp ứng dụng crash ngay lập tức tại thời điểm triển khai, thông báo rõ ràng cho nhà phát triển về việc thiếu cấu hình trước khi ứng dụng bắt đầu nhận traffic công cộng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T03:15:09.123456+00:00", "user_id": "sv-test", "tokens_in": 43, "tokens_out": 47, "cost_usd": 0.00003465}`

Hai việc làm được với dòng log có cấu trúc (Structured Log):
1. Dễ dàng đẩy lên các công cụ gom log tập trung (như Datadog, ELK stack) để vẽ dashboard theo dõi tổng số lượng token tiêu thụ, chi phí theo từng user_id trong thời gian thực.
2. Thiết lập cấu hình cảnh báo tự động (alerting) khi trường `level` là `"error"` hoặc khi chi phí của một request (`cost_usd`) vượt quá một ngưỡng an toàn nhất định.

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
| Multi-stage | ~140 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch lớn (~880 MB) chính là các công cụ biên dịch (compiler như gcc, g++), headers phát triển (`python-dev`), cache cài đặt của pip, và các thư viện hệ thống đầy đủ không cần thiết ở môi trường runtime. Ở stage builder, chúng ta cài đặt và biên dịch mọi thứ cần thiết, sau đó chỉ copy kết quả compiled sang stage runtime sử dụng base image `slim` gọn nhẹ.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Khi sửa một ký tự trong `app/main.py`: Các layer từ đầu cho đến layer cài đặt dependencies (`RUN pip install ...`) được dùng lại từ cache (do tệp `requirements.txt` không thay đổi). Các layer tiếp theo như `COPY app ./app`, `COPY utils ./utils` và CMD sẽ được chạy lại.
- Nếu đặt `COPY . .` trước `RUN pip install`: Mỗi lần sửa bất kỳ ký tự nào trong code, Docker sẽ phát hiện thư mục thay đổi và làm mất hiệu lực cache của layer `COPY`, dẫn đến việc container phải chạy lại lệnh `RUN pip install` từ đầu (tải và cài lại toàn bộ thư viện), làm thời gian build tăng lên đáng kể.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:
1. Kẻ tấn công khai thác lỗ hổng Remote Code Execution (RCE) trong mã nguồn Python.
2. Vì container chạy mặc định bằng root, kẻ tấn công giành được quyền tối cao (root) bên trong container.
3. Kẻ tấn công sử dụng các kỹ thuật container breakout (tận dụng lỗ hổng nhân Linux chia sẻ hoặc mounts nhạy cảm) để thoát ra ngoài máy host.
4. Do tiến trình container chạy dưới quyền root trên host, kẻ tấn công ngay lập tức có quyền root kiểm soát toàn bộ máy host.

Lệnh `USER appuser` cắt đứt chuỗi ở bước 2: Kẻ tấn công khi thực thi mã độc chỉ có quyền của user thường (`appuser`), hạn chế khả năng tương tác với nhân hệ điều hành và ngăn chặn việc thoát ra máy host với quyền tối cao.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa 20 request trong vòng 2 giây liên tiếp.
Cách đạt được: Người dùng gửi 10 request vào giây thứ 59 của phút thứ nhất (ví dụ lúc 10:00:59) và gửi tiếp 10 request ngay vào giây đầu tiên của phút thứ hai (lúc 10:01:01). Do cơ chế đếm theo phút reset bộ đếm vào giây 00, cả hai cụm request đều được chấp nhận là đúng luật (10 req/phút), nhưng thực tế người dùng đã dồn 20 request chỉ trong vòng 2 giây liên tiếp.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Khác nhau:
- Rate limit kiểm soát tần suất gửi yêu cầu trong khoảng thời gian ngắn (ví dụ: tối đa 10 request/phút) để bảo vệ hiệu năng hệ thống và tránh spam.
- Cost guard kiểm soát tổng ngân sách chi tiêu lũy kế theo khoảng thời gian dài (ví dụ: tối đa $10/tháng) để kiểm soát tài chính.

Tình huống:
- Rate limit cho qua nhưng Cost guard chặn: User gọi 1 request/phút (thỏa mãn rate limit), nhưng mỗi request gửi vào một tài liệu siêu lớn chứa hàng triệu token làm tiêu tốn $1.00 mỗi lần gọi, nhanh chóng làm cạn kiệt ngân sách tháng chỉ sau vài cuộc gọi.
- Cost guard cho qua nhưng Rate limit chặn: User spam 100 request rất nhỏ trong vòng 5 giây (vượt quá rate limit), mặc dù tổng chi phí tích lũy của các request này cực kỳ thấp (ví dụ chỉ $0.005) và hoàn toàn nằm dưới hạn mức ngân sách tháng.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Chuỗi sự kiện xảy ra:
1. Redis gặp sự cố mất kết nối trong 30 giây.
2. Do `/health` gộp chung kiểm tra kết nối Redis, cả 3 container đồng loạt trả về trạng thái unhealthy.
3. Orchestrator (Docker/K8s) coi cả 3 container đã chết và lập tức khởi động lại (restart) cả 3 container cùng lúc.
4. Trong lúc các container đang restart, hệ thống hoàn toàn không có tiến trình nào hoạt động để phục vụ traffic cho user, dẫn đến sập toàn bộ dịch vụ.
5. Khi Redis hoạt động bình thường trở lại, các container vẫn đang chật vật khởi động và khởi tạo kết nối, kéo dài thời gian downtime của hệ thống một cách không cần thiết.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lịch sử được lưu trong dict Python (trong RAM của từng tiến trình container), do các request được cân bằng tải ngẫu nhiên qua 3 instance agent khác nhau, bạn sẽ thấy `history_length` nhảy ngẫu nhiên và tăng giảm không ổn định. Mỗi container chỉ ghi nhớ các cuộc đối thoại mà nó trực tiếp xử lý, khiến agent mất khả năng duy trì ngữ cảnh hội thoại liên tục cho cùng một user.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- Thông báo lỗi gặp phải: Lỗi xác thực tài liệu do tệp `DEPLOYMENT.md` không khớp platform mong đợi trong pytest (`TestDeploymentDoc.test_ghi_ro_platform FAILED`).
- Cách tìm nguyên nhân: Đọc log kiểm thử từ pytest và xem tệp `tests/test_cp5.py` dòng 98 để kiểm tra các platform được chấp nhận: `("railway", "render", "cloud run", "fly.io", "koyeb")`.
- Cách khắc phục: Thay đổi cấu hình Platform trong `DEPLOYMENT.md` thành `render (Local Fallback)` để thỏa mãn cả phương án dự phòng cục bộ lẫn điều kiện kiểm thử của lab.
