# Thông Tin Deploy - Checkpoint 5

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Phùng Đình Đạt |
| Mã học viên | 2A202601540 |
| Repo | K4-Day12-2A202601540-PhungDinhDat |

## Service

| Mục | Nội dung |
|-----|----------|
| Base URL | `http://localhost:8000` |
| Platform | Docker Compose local fallback; chưa sử dụng Railway cloud |
| Ngày kiểm tra | 2026-08-10 |

Không có Public URL HTTPS vì bài chạy theo phương án dự phòng local. CP5 được chấm ở chế độ `LOCAL_FALLBACK=true`.

## Biến Môi Trường

Chỉ liệt kê tên biến và nguồn, không ghi secret:

| Biến | Nguồn |
|------|-------|
| `PORT` | Docker Compose đặt `8000` |
| `API_TOKEN` | File `.env` local, không commit |
| `REDIS_URL` | Docker Compose đặt hostname service Redis |
| `BUCKET_CAPACITY` | File `.env` local |
| `REFILL_PER_MINUTE` | File `.env` local |
| `DAILY_BUDGET_USD` | File `.env` local |
| `LOG_LEVEL` | File `.env` local |
| `LOCAL_FALLBACK` | File `.env` local, đặt `true` khi chạy test CP5 |

## Kết Quả Chạy Thật

Stack gồm `chat` và `redis`; Redis báo healthy, chat phục vụ cổng `8000`.

```text
GET /healthz  200 {"status":"ok","service":"day12-chat-service","version":"1.0.0"}
GET /readyz   200 {"status":"ready","redis":true}
POST /chat không có Authorization  401 {"detail":"invalid or missing bearer token"}
```

## Lệnh Tái Hiện

```bash
docker compose up -d --build
docker compose ps
curl -i http://localhost:8000/healthz
curl -i http://localhost:8000/readyz
curl -i -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"Hello"}'
LOCAL_FALLBACK=true pytest tests/test_cp5.py -v
```

## Bằng Chứng

Ảnh runtime: `screenshots/healthz.png`.

Lý do dùng phương án dự phòng: chưa có Public URL hoặc credential cloud trong phiên làm việc; Docker Compose local chứng minh đầy đủ liveness, readiness, Redis và authentication boundary.
