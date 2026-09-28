# Thông Tin Deploy — Checkpoint 5

> `pytest tests/test_cp5.py` đọc file này để tìm địa chỉ service và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, không dán giá trị API key vào đây.**

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyễn Xuân Trường Giang |
| Mã học viên | 2A202602446 |
| Repo | https://github.com/GiangDA881/K4-L3A-DAY12-NguyenXuanTruongGiang-2A202602446-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://agent-production-ea27.up.railway.app |
| Platform | Railway (project `day12-agent`, service `agent` + service `Redis`) |
| Ngày deploy | 2026-09-28 |
| Cách deploy | `railway up --service agent` — build từ `Dockerfile` multi-stage trong repo |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | Railway tự gán; `CMD` trong Dockerfile đọc `${PORT:-8000}` |
| `AGENT_API_KEY` | ✅ | đặt trong Railway Variables của service `agent`, không nằm trong repo |
| `REDIS_URL` | ✅ | biến tham chiếu `${{Redis.REDIS_URL}}` tới Redis add-on của Railway (mạng nội bộ) |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

`URL=https://agent-production-ea27.up.railway.app`

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i $URL/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i $URL/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST $URL/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST $URL/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST $URL/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Chạy ngày 2026-09-28 từ máy cá nhân (Windows, Git Bash) vào bản deploy trên Railway:

```
== 1 health
HTTP/1.1 200 OK
{"status":"ok","service":"day12-agent","version":"1.0.0"}

== 2 ready
HTTP/1.1 200 OK
{"status":"ready","redis":true}

== 3 no key
HTTP/1.1 401 Unauthorized
{"detail":"invalid or missing API key"}

== 4 with key  (gửi bằng Python httpx — xem ghi chú bên dưới)
HTTP 200
{"answer":"Câu hỏi hay. Deploy là gì thường được giải quyết bằng cách chuẩn hóa môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud. (Mình đang nhớ 2 lượt trao đổi trước đó.)","user_id":"sv-deploy","history_length":2,"cost_usd":3.315e-05,"tokens":{"in":41,"out":45}}

== 5 rate limit
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

Ghi chú lệnh 4: gọi bằng `curl` trong Git Bash trên Windows với body có
tiếng Việt (`"Deploy là gì?"`) trả `400 {"detail":"There was an error parsing
the body"}` — terminal gửi chuỗi không phải UTF-8 nên FastAPI không parse được
JSON. Không phải lỗi service: gửi lại body UTF-8 bằng Python `httpx` thì trả 200
như trên.

Log runtime trên Railway (`railway logs --service agent`) — Railway tự tách log
JSON thành các trường có cấu trúc:

```
[INFO]  event="ask_completed" timestamp="2026-09-28T08:21:59.225213+00:00" user_id="sv-test" tokens_in=392 tokens_out=43 cost_usd=0.0000846
INFO:     100.64.0.9:44208 - "POST /ask HTTP/1.1" 429 Too Many Requests
[INFO]  event="ask_completed" timestamp="2026-09-28T08:22:27.967106+00:00" user_id="sv-deploy" tokens_in=41 tokens_out=45 cost_usd=0.00003315
```

## Ảnh Chụp Màn Hình

Thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang project `day12-agent` trên Railway (service `agent` + `Redis`)
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt
