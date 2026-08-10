# Thông Tin Deploy — Checkpoint 5

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyễn Chí Hướng |
| Mã học viên | 2A202601203 |
| Repo | https://github.com/Huong-cheseway/K3_DAY12_2A202601203_NguyenChiHuong |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://agent-production-ddfe.up.railway.app |
| Platform | Railway |
| Project | K3-Day12-2A202601203-NguyenChiHuong |
| Ngày deploy | 2026-08-10 |

## Biến Môi Trường Đã Set Trên Cloud

Chỉ liệt kê tên và nguồn của biến; giá trị bí mật không nằm trong tài liệu này.

| Biến | Đã set | Nguồn |
|------|--------|-------|
| `PORT` | ✅ | Railway tự gán |
| `AGENT_API_KEY` | ✅ | Railway service variable (secret) |
| `REDIS_URL` | ✅ | Tham chiếu service Redis của Railway |
| `RATE_LIMIT_PER_MINUTE` | ✅ | Railway service variable |
| `MONTHLY_BUDGET_USD` | ✅ | Railway service variable |
| `LOG_LEVEL` | ✅ | Railway service variable |

## Kết Quả Kiểm Tra Bản Deploy

```text
GET /health
200 {"status":"ok","service":"day12-agent","version":"1.0.0"}

GET /ready
200 {"status":"ready","redis":true}

POST /ask (không gửi X-API-Key)
401 Unauthorized

POST /ask (gửi X-API-Key hợp lệ)
200 OK, user_id=cp5-smoke, answer_present=true, history_length=0
```

## Lệnh Kiểm Tra

```bash
URL=https://agent-production-ddfe.up.railway.app

curl -i "$URL/health"
curl -i "$URL/ready"
curl -i -X POST "$URL/ask" \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# AGENT_API_KEY được lấy từ môi trường local, không ghi trong repo.
curl -i -X POST "$URL/ask" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'
```

## Ảnh Chụp Màn Hình

- `screenshots/dashboard.png`: Railway project và hai service `agent`, `Redis`.
- `screenshots/health.png`: kết quả gọi endpoint `/health`.
