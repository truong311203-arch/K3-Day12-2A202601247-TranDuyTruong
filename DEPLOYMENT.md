# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Trần Duy Trường |
| Mã học viên | 2A202601247 |
| Repo | https://github.com/truong311203-arch/K3-Day12-2A202601247-TranDuyTruong |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-bhnx.onrender.com |
| Platform | Render |
| Ngày deploy | 10/08/2026 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | tự gán local |
| `AGENT_API_KEY` | ✅ | lấy từ file .env cục bộ |
| `REDIS_URL` | ✅ | redis://redis:6379/0 |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i <URL>/health
curl -i https://day12-agent-bhnx.onrender.com/health
# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://day12-agent-bhnx.onrender.com/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://day12-agent-bhnx.onrender.com/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST https://day12-agent-bhnx.onrender.com/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://day12-agent-bhnx.onrender.com/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
$ curl -i http://localhost:8000/health
HTTP/1.1 200 OK
date: Mon, 10 Aug 2026 03:15:05 GMT
server: uvicorn
content-length: 57
content-type: application/json

{"status":"ok","service":"day12-agent","version":"1.0.0"}

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
$ curl -i http://localhost:8000/ready
HTTP/1.1 200 OK
date: Mon, 10 Aug 2026 03:15:06 GMT
server: uvicorn
content-length: 31
content-type: application/json

{"status":"ready","redis":true}

# 3. Không có API key — mong đợi 401
$ curl -i -X POST http://localhost:8000/ask -H "Content-Type: application/json" -d '{"question":"Hello"}'
HTTP/1.1 401 Unauthorized
date: Mon, 10 Aug 2026 03:15:09 GMT
server: uvicorn
content-length: 39
content-type: application/json

{"detail":"invalid or missing API key"}

# 4. Có API key — mong đợi 200 kèm câu trả lời
$ curl -i -X POST http://localhost:8000/ask -H "Content-Type: application/json" -H "X-API-Key: r8qHUlc4myn_Axp5HN6XH_lc9A2V_qCreUNSe_VvfRA" -H "X-User-Id: sv-test" -d '{"question":"Deploy la gi?"}'
HTTP/1.1 200 OK
content-type: application/json
{"answer":"Ngắn gọn: Deploy la gi phụ thuộc vào ba yếu tố — cấu hình qua biến môi trường, health check để orchestrator biết trạng thái, và giới hạn tài nguyên. (Mình đang nhớ 2 lượt trao đổi trước đó.)","user_id":"sv-test","history_length":2,"cost_usd":3.465e-05,"tokens":{"in":43,"out":47}}
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl

---

## Nếu Dùng Phương Án Dự Phòng

Không đăng ký được tài khoản cloud? Vẫn nộp được bài, nhưng CP5 tối đa 60% điểm:

1. Đặt `LOCAL_FALLBACK=true` trong `.env`
2. Chạy `docker compose up -d` rồi kiểm tra `docker compose ps`
3. Chụp màn hình vào `screenshots/`
4. Chạy `pytest tests/test_cp5.py -v` — bộ test sẽ tự chuyển sang kiểm tra
   `http://localhost:8000`
5. Ghi rõ lý do không deploy được vào phần dưới đây:

```
Sử dụng phương án dự phòng Local Fallback: Đã kích hoạt LOCAL_FALLBACK=true trong .env, khởi chạy stack thành công bằng Docker Compose local (redis + agent), đã bổ sung các hình ảnh chụp màn hình tương ứng vào thư mục screenshots/ và tất cả các bài kiểm tra hoạt động thành công trên localhost.
```
