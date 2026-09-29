# Thông Tin Deploy — Checkpoint 5

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Lê Trọng Khánh |
| Mã học viên | 2A202602941 |
| Repo | https://github.com/khanhlt2611/K4-L3B-DAY12-LeTrongKhanh-2A202602941-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://agent-production-aa8b.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-09-29 |

Project Railway gồm web service `agent` được build bằng Dockerfile và một
Redis service có persistent volume. Public domain dùng HTTPS và chuyển tiếp
traffic tới cổng do Railway cấp cho container.

## Biến Môi Trường Đã Set Trên Cloud

Chỉ liệt kê tên biến và nguồn giá trị; giá trị secret không được lưu trong repo.

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | Railway tự gán |
| `AGENT_API_KEY` | ✅ | secret đặt bằng Railway CLI |
| `REDIS_URL` | ✅ | tham chiếu `Redis.REDIS_URL` của Redis service trong cùng project |
| `RATE_LIMIT_PER_MINUTE` | ✅ | cấu hình giới hạn request |
| `MONTHLY_BUDGET_USD` | ✅ | cấu hình ngân sách |
| `LOG_LEVEL` | ✅ | mức log production |

## Lệnh Kiểm Tra

```bash
# Liveness
curl -i https://agent-production-aa8b.up.railway.app/health

# Readiness và kết nối Redis
curl -i https://agent-production-aa8b.up.railway.app/ready

# Không có API key, mong đợi 401
curl -i -X POST https://agent-production-aa8b.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# Có API key, mong đợi 200; giá trị key lấy từ môi trường cục bộ
curl -i -X POST https://agent-production-aa8b.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: cp5-test" \
  -d '{"question":"Deploy là gì?"}'
```

## Kết Quả Chạy Thật

Kết quả xác minh sau khi deployment và Redis cùng ở trạng thái `SUCCESS`:

```text
GET  /health                 200  {"status":"ok","service":"day12-agent","version":"1.0.0"}
GET  /ready                  200  {"status":"ready","redis":true}
POST /ask (không API key)    401  {"detail":"invalid or missing API key"}
POST /ask (có API key)       200  response có answer, token usage và cost_usd
```

## Ảnh Chụp Màn Hình

- `screenshots/Railway.png` — dashboard cho thấy `agent` và Redis đều online,
  deployment Redis thành công và persistent volume đã được gắn.
- `screenshots/health.png` — endpoint `/health` của public service.
