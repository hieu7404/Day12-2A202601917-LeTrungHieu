# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lê Trung Hiếu.  Mã học viên: 2A202601917.

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu để mặc định là "changeme", khi đưa ứng dụng lên môi trường thực tế mà quên cấu hình key thật, ứng dụng vẫn sẽ hoạt động bình thường. Lỗ hổng là kẻ xấu có thể dùng key mặc định này để gọi API miễn phí và gây thiệt hại lớn về tài chính. Việc ứng dụng báo lỗi và dừng ngay lập tức (fail-fast) bắt buộc lập trình viên phải thiết lập key hợp lệ trước khi chạy, giúp ngăn chặn rủi ro này từ đầu.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON: {"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T05:47:42.174621+00:00", "user_id": "sv-123", "tokens_in": 3, "tokens_out": 41, "cost_usd": 2.505e-05}
Nhờ log này mình làm được 2 việc:
- Dùng các công cụ giám sát để tự động đếm và tính tổng chi phí (cost_usd) mà từng user đã sử dụng trong tháng.
- Dễ dàng dùng câu lệnh truy vấn (query) để lọc ra các request có số lượng tokens_out > 1000 nhằm mục đích tối ưu hóa, điều mà hàm print() thông thường không làm được vì thiếu dữ liệu định lượng.

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
| 1 stage (bản đầu) | ... MB |
| Multi-stage | ... MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> | Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | ~1.73GB |
| Multi-stage | ~270MB |

Giải thích: Phần chênh lệch dung lượng hơn 1.4 GB đó là do các trình biên dịch hệ thống (như gcc) và mã nguồn C++ dư thừa sinh ra trong lúc tải thư viện. Ở bản Multi-stage, chúng ta chỉ sao chép các file thư viện đã được biên dịch hoàn chỉnh sang môi trường chạy thực tế (stage 2), bỏ lại những thứ không cần thiết.
---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi sửa main.py: Các bước COPY requirements.txt và RUN pip install vẫn sử dụng lại bộ nhớ đệm (cache) cũ nên chạy rất nhanh. Chỉ từ lệnh COPY app trở đi mới phải thực thi lại.
Nếu đặt COPY . . lên trên cùng: Mỗi khi sửa đổi bất kỳ nội dung nào trong code, Docker sẽ làm mất cache từ dòng lệnh đó. Dẫn đến việc lệnh pip install phía dưới cũng bị chạy lại toàn bộ, gây lãng phí nhiều phút đồng hồ để tải lại thư viện.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Container mặc định chạy bằng quyền root. Nếu ứng dụng Python có lỗ hổng (ví dụ: cho phép thực thi mã lệnh từ xa), kẻ tấn công sẽ chiếm được quyền root bên trong container, từ đó có thể tìm cách thoát ra ngoài (container breakout) để phá hoại máy chủ thật. Lệnh USER appuser ép ứng dụng chạy dưới quyền người dùng thông thường, giới hạn hoàn toàn đặc quyền của kẻ tấn công ngay cả khi chúng xâm nhập được vào container.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Một người dùng có thể gửi tối đa 20 request trong 2 giây liên tiếp.
Giải thích: Nếu đếm theo phút, người dùng gửi 10 request vào lúc 10:00:59 (thuộc phút cũ nên hợp lệ). Sau đó đúng 1 giây, họ gửi tiếp 10 request vào lúc 10:01:00 (đã sang phút mới, hệ thống reset bộ đếm về 0 nên tiếp tục hợp lệ). Do đó, máy chủ phải chịu 20 request chỉ trong vòng 2 giây, làm mất đi tác dụng giới hạn tải của Rate Limit.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Sự khác biệt: Rate limit giới hạn "số lượng/tần suất gọi", còn Cost guard giới hạn "tổng chi phí/ngân sách".
- Rate limit cho qua, Cost guard chặn: User lần đầu gọi API trong tháng (thoát Rate limit) nhưng gửi một đoạn văn bản rất dài tiêu tốn 15$ (vượt ngân sách 10$) ➔ Bị Cost guard chặn.
- Cost guard cho qua, Rate limit chặn: User mới tiêu hết 0.1$ (thoát Cost guard) nhưng cài bot gọi liên tục 15 câu chat siêu ngắn trong vòng 1 phút ➔ Rate limit sẽ chặn lại từ câu thứ 11.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Trình tự sự kiện nếu gộp chung:
- Redis mất kết nối mạng 30 giây.
- Hàm /health (vì đã gộp chung lệnh kiểm tra Redis) lập tức báo lỗi 503.
- Hệ thống quản trị (như Docker/Kubernetes) gọi vào /health thấy lỗi liền lầm tưởng cả 3 container ứng dụng Python đều đã bị treo/hỏng.
- Hệ thống thẳng tay khởi động lại (restart) cả 3 container cùng một lúc, gây gián đoạn toàn bộ dịch vụ.
(Nếu tách riêng: /ready báo 503 để ngừng nhận request mới chờ Redis hồi phục, nhưng /health vẫn báo 200 do Python vẫn chạy, hệ thống sẽ không bị restart oan uổng).


---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Nếu lưu lịch sử bằng biến toàn cục (dict) trong RAM thay vì dùng chung Redis:
Mỗi container có một vùng RAM hoàn toàn độc lập. Khi chạy 3 container, Load Balancer sẽ phân phối các request ngẫu nhiên (VD: Câu 1 vào máy A, câu 2 vào máy B).
Kết quả: Sẽ thấy thông số history_length nhảy loạn xạ (lúc thì 1, lúc lại về 1, lúc thì 2) thay vì tăng dần đều. Nguyên nhân do mỗi máy chủ chỉ lưu được đúng phần hội thoại mà chính nó xử lý và hoàn toàn không biết gì về các đoạn chat đã gửi cho 2 máy còn lại.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi gặp phải: Truy cập vào Public URL trên trình duyệt nhận được kết quả {"detail": "Not Found"}. Thông báo lỗi: Lỗi 404 Not Found. Nguyên nhân: Do theo thói quen click trực tiếp vào đường link URL gốc của ứng dụng (ví dụ: https://day12-agent-x71y.onrender.com/) trên dashboard. Tuy nhiên, ứng dụng FastAPI này không định nghĩa endpoint nào ở thư mục gốc /. Cách sửa: Gõ thêm đuôi /health vào thanh địa chỉ của trình duyệt (https://.../health) để truy cập đúng endpoint kiểm tra liveness, kết quả trả về {"status":"ok",...} thành công.
