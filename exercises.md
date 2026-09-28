# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Họ và tên: Nguyễn Xuân Trường Giang  Mã học viên: 2A202602446

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Railway, service `agent` được tạo riêng với Redis. Nếu mình
> quên set `AGENT_API_KEY` ở tab Variables mà code có mặc định `"changeme"`,
> container vẫn lên, `/health` vẫn 200, Railway báo deploy thành công — nhưng
> `/ask` trên URL công khai giờ mở cho bất kỳ ai đoán được `changeme` (một chuỗi
> nằm ngay trong repo public). Bot quét Internet gọi vào và mình chỉ biết khi
> xem hóa đơn LLM. Không có mặc định thì `Settings()` ném `ValidationError:
> agent_api_key Field required` ngay lúc khởi động, healthcheck của Railway
> fail, deploy bị đánh đỏ và lỗi hiện rõ trong log — mình sửa khi còn đang
> nhìn màn hình, trước khi có request nào lọt vào. Test
> `test_thieu_api_key_thi_fail_fast` kiểm tra đúng hành vi này.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thật khi chạy `uvicorn` ở máy:
>
> ```
> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T08:09:11.916752+00:00", "user_id": "sv01", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}
> ```
>
> 1. **Lọc và cộng dồn theo trường**: gom các dòng `event="ask_completed"`
>    rồi cộng `cost_usd` theo `user_id` để biết user nào tiêu nhiều tiền nhất
>    hôm nay. Trên Railway mình thấy log được tự tách thành các trường
>    (`event="ask_completed" user_id="sv-test" cost_usd=0.0000846`) nên lọc
>    được ngay trên dashboard.
> 2. **Đặt cảnh báo / tính tỷ lệ theo thời gian**: dựa vào `timestamp` và
>    `level` để đếm số `level="error"` trong 5 phút qua, hoặc cảnh báo khi
>    tổng `cost_usd` trong 1 giờ vượt ngưỡng. Với `print("đã trả lời xong")`
>    chỉ có một câu chữ tự do — không biết của user nào, tốn bao nhiêu, lúc
>    nào, nên máy không đếm hay lọc được.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f Dockerfile.single -t agent:single .
docker build -t day12-agent:prod .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | 1730 MB (1.73GB) |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chênh ~1.46GB, gần như toàn bộ đến từ **base image**: `python:3.11` bản đầy
> đủ dựa trên Debian có sẵn gcc, make, các header `-dev`, git, curl... để
> biên dịch được mọi thứ; `python:3.11-slim` bỏ hết những thứ đó. `docker
> history agent:single` cho thấy layer `pip install` chỉ 95.1MB và `COPY . .`
> chỉ 147kB — tức phần code và thư viện của mình rất nhỏ, còn lại là hệ điều
> hành. Multi-stage giúp thêm ở chỗ: stage `builder` cài dependency vào
> `/install` (kèm cache pip, wheel tạm), stage `runtime` chỉ `COPY --from=builder
> /install` sang, nên mọi thứ phục vụ việc build bị vứt lại. Ngoài ra
> `.dockerignore` loại `.git`, `.venv`, `tests`, `.env` khỏi build context.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Mình thêm một dòng comment vào cuối `app/main.py` rồi `docker build
> --progress=plain`. Kết quả: các bước `WORKDIR /build`, `COPY
> requirements.txt`, `RUN pip install ... --prefix=/install` (builder) và
> `RUN useradd`, `WORKDIR /app`, `COPY --from=builder /install` (runtime) đều
> báo `CACHED`; chỉ `COPY app ./app` và `COPY utils ./utils` chạy lại. Cả lần
> build mất **4.2 giây**.
>
> Nếu đặt `COPY . .` trước `pip install` thì checksum của layer COPY đổi mỗi
> khi sửa bất kỳ file nào, Docker huỷ cache từ layer đó trở xuống → `pip
> install` chạy lại toàn bộ, tải lại mọi package (lần build đầu của mình mất
> vài phút vì mạng chậm). Mỗi lần sửa một dấu phẩy là phải chờ lâu như vậy.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> 1. Code có lỗ hổng (ví dụ một thư viện parse input bị RCE, hoặc mình lỡ
>    `eval` dữ liệu người dùng) → kẻ tấn công chạy được lệnh shell **với
>    quyền của process uvicorn**.
> 2. Nếu process là root (uid 0) trong container, họ đọc/ghi được mọi file
>    trong container, cài thêm công cụ, sửa code của app.
> 3. uid 0 trong container là uid 0 trên kernel của host (không có user
>    namespace). Chỉ cần một cấu hình lỏng — volume mount thư mục host,
>    `docker.sock` bị mount vào, `--privileged`, hay một lỗ hổng kernel để
>    thoát container — là họ thành root trên host.
>
> `USER appuser` (uid 10001) cắt chuỗi ở bước 2: shell của kẻ tấn công chỉ là
> user thường, không cài được package, không ghi được vào `/usr/local`, và nếu
> có thoát ra host thì cũng chỉ là uid 10001 không có quyền gì. Mình đã kiểm
> tra: `docker compose exec agent id` → `uid=10001(appuser) gid=10001(appuser)`.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> **20 request.** Gửi 10 request lúc 10:00:59 — hết quota của phút 10:00. Đến
> 10:01:00 bộ đếm reset về 0, gửi tiếp 10 request lúc 10:01:00–10:01:01. Cả
> 20 request đều "đúng luật" dù nằm trong ~2 giây, gấp đôi hạn mức.
>
> Với sliding window, lúc 10:01:01 mình đếm các request trong khoảng
> (10:00:01, 10:01:01] — 10 request lúc 10:00:59 vẫn còn trong cửa sổ, nên
> request thứ 11 bị 429 ngay. Trên bản deploy mình gọi liên tiếp 15 lần và
> nhận đúng `200 ×10` rồi `429 ×5`.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn **số request trong 60 giây** (tốc độ, chống spam/bot,
> trả 429, tự hồi sau 1 phút). Cost guard giới hạn **tổng tiền trong tháng**
> theo user (trả 402, chỉ reset khi sang tháng mới vì key là
> `cost:<user>:<YYYY-MM>`).
>
> - Rate limit cho qua, cost guard chặn: user gửi đều đặn 5 request/phút
>   (dưới hạn mức 10) nhưng mỗi câu hỏi dán kèm tài liệu dài 50.000 token,
>   chạy suốt nhiều ngày — tốc độ bình thường nhưng tiền cộng dồn vượt 10
>   USD → cost guard trả 402.
> - Cost guard cho qua, rate limit chặn: một script lỗi gọi `/ask` với câu
>   hỏi "hi" 100 lần trong vài giây — mỗi lần chỉ tốn ~0.00002 USD nên còn xa
>   ngân sách, nhưng request thứ 11 trong phút đã bị 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> 1. t=0: Redis mất kết nối. Cả 3 container đều gọi `ping()` thất bại → cả 3
>    endpoint health trả 503 cùng lúc.
> 2. Orchestrator coi 503 ở liveness là "process hỏng" → sau vài lần retry
>    (ví dụ 3 × 10s), nó **restart cả 3 container** gần như đồng thời.
> 3. Trong lúc restart, không còn instance nào nhận request — kể cả những
>    request không cần Redis — user nhận 502/503 hàng loạt.
> 4. t=30s: Redis quay lại, nhưng các container vẫn đang khởi động (có thể
>    đang bị restart vòng thứ hai vì health vừa fail) → downtime kéo dài hơn
>    chính sự cố Redis; container mới lên cùng lúc còn tạo "thundering herd"
>    đập vào Redis.
>
> Tách ra thì: `/ready` trả 503 → load balancer chỉ **ngừng gửi traffic**,
> container vẫn sống; `/health` vẫn 200 nên không bị restart. Redis quay lại
> → `/ready` 200 → traffic chảy lại ngay, không mất thời gian khởi động.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Mình scale 3 container (cổng 8000, 8001, 8002) và gọi lần lượt từng
> container với cùng `X-User-Id: scale-test`:
>
> ```
> port 8000 -> history_length=0
> port 8001 -> history_length=2
> port 8002 -> history_length=4
> port 8000 -> history_length=6
> port 8001 -> history_length=8
> ```
>
> Con số tăng đều 2 mỗi lượt dù mỗi lượt vào container khác, và
> `redis-cli LLEN history:scale-test` = 10 — cả 3 container đọc/ghi cùng
> một list trong Redis.
>
> Nếu dùng dict trong RAM, mỗi container có dict riêng nên sẽ thấy
> `0, 0, 0, 2, 2` — lượt 2 và 3 vào container chưa từng thấy user này nên
> agent "quên" hết; con số nhảy lung tung tùy load balancer gửi vào đâu, và
> reset về 0 mỗi khi container bị restart/deploy lại.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> **Lỗi:** sau khi deploy lên Railway, lệnh kiểm tra số 4 (gọi `/ask` có key
> bằng `curl` trong Git Bash trên Windows) trả
> `HTTP/1.1 400 Bad Request {"detail":"There was an error parsing the body"}`,
> trong khi `/health`, `/ready` đều 200, gọi không key trả 401 đúng, và vòng
> lặp 15 lần với body `{"question":"test"}` vẫn trả 200/429 bình thường.
>
> **Tìm nguyên nhân:** vì `/ready` 200 và body chỉ có chữ ASCII chạy được,
> mình loại trừ Redis, API key và code của service. Khác biệt duy nhất là
> body có tiếng Việt (`"Deploy là gì?"`), và lỗi 400 xảy ra ở bước FastAPI
> parse JSON — trước cả khi vào hàm `ask`. Gửi đúng câu đó bằng Python
> `httpx` (luôn mã hóa JSON thành UTF-8) thì nhận 200. Vậy lỗi nằm ở phía
> client: terminal Windows gửi chuỗi theo code page cũ chứ không phải UTF-8,
> server nhận byte không hợp lệ.
>
> **Sửa:** gửi body dạng UTF-8 (dùng `httpx`/`--data-binary @file.json` với
> file lưu UTF-8, hoặc chỉ dùng ASCII khi test bằng curl trên Windows). Service
> không phải sửa gì. Ngoài ra, trước khi deploy mình đã bỏ `startCommand` trong
> `railway.toml`, vì Railway chạy lệnh đó dạng exec nên `$PORT` không được
> thay; để `CMD ["sh", "-c", "... --port ${PORT:-8000}"]` trong Dockerfile lo
> việc này, và healthcheck `/health` pass ngay lần deploy đầu.
