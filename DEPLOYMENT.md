# Thông Tin Deploy - Checkpoint 5

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Phùng Đình Đạt |
| Mã học viên | 2A202601540 |
| Repo | K4-Day12-2A202601540-PhungDinhDat |

## Service Render

| Mục | Nội dung |
|-----|----------|
| Public URL | Chờ Render cấp domain sau lần deploy đầu tiên |
| Platform | Render Blueprint |
| Blueprint | `render.yaml` |
| Ngày chuẩn bị | 2026-08-10 |

Sau khi Render deploy thành công, thay trạng thái Public URL bằng domain HTTPS thật dạng `https://<service>.onrender.com` rồi chạy lại `tests/test_cp5.py`.

## Tài Nguyên Blueprint

`render.yaml` tạo hai service:

| Service | Loại | Vai trò |
|---------|------|---------|
| `day12-chat` | Docker web service | Chạy FastAPI từ `Dockerfile`; health check tại `/healthz` |
| `day12-chat-redis` | Render Key Value | Lưu history, token bucket và daily cost qua Redis protocol |

`REDIS_URL` được Render inject từ `day12-chat-redis.connectionString`; không sao chép connection string vào repo.

## Biến Môi Trường

| Biến | Cách cấu hình trên Render |
|------|---------------------------|
| `PORT` | Render tự cấp cho web service |
| `API_TOKEN` | Render yêu cầu nhập khi tạo Blueprint vì `sync: false` |
| `REDIS_URL` | Blueprint reference tới Render Key Value |
| `BUCKET_CAPACITY` | Blueprint đặt `10` |
| `REFILL_PER_MINUTE` | Blueprint đặt `10` |
| `DAILY_BUDGET_USD` | Blueprint đặt `1.0` |
| `LOG_LEVEL` | Blueprint đặt `INFO` |

Không ghi giá trị `API_TOKEN` hoặc `REDIS_URL` vào tài liệu, Git hay screenshot.

## Các Bước Deploy

1. Push branch chứa `render.yaml` lên GitHub.
2. Trong Render Dashboard, chọn **New > Blueprint** và kết nối repository.
3. Chọn branch cần deploy; Render đọc `render.yaml` ở repo root.
4. Nhập giá trị bí mật cho `API_TOKEN` khi Render hỏi.
5. Apply Blueprint; đợi `day12-chat-redis` và `day12-chat` hoạt động.
6. Mở domain Render và kiểm tra `/healthz`, `/readyz`, `/chat`.
7. Cập nhật Public URL thật trong tài liệu này.

## Lệnh Kiểm Tra Cloud

```bash
export BASE_URL='https://<service>.onrender.com'
export DEPLOY_API_TOKEN='<cùng giá trị API_TOKEN đã nhập trên Render>'

curl -i "$BASE_URL/healthz"
curl -i "$BASE_URL/readyz"
curl -i -X POST "$BASE_URL/chat" \
  -H "Content-Type: application/json" \
  -d '{"message":"Hello"}'
curl -i -X POST "$BASE_URL/chat" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $DEPLOY_API_TOKEN" \
  -H "X-Client-Id: cp5-test" \
  -d '{"message":"Deploy là gì?"}'

DEPLOY_API_TOKEN="$DEPLOY_API_TOKEN" pytest tests/test_cp5.py -v
```

Kết quả yêu cầu:

```text
GET /healthz                         200
GET /readyz                          200, redis=true
POST /chat không có Authorization    401
POST /chat có Bearer token hợp lệ    200
```

## Bằng Chứng Local Trước Deploy

Docker Compose đã xác minh cùng image và API contract trước khi chuyển sang Render:

```text
GET /healthz  200 {"status":"ok","service":"day12-chat-service","version":"1.0.0"}
GET /readyz   200 {"status":"ready","redis":true}
POST /chat không có Authorization  401 {"detail":"invalid or missing bearer token"}
```

Ảnh local: `screenshots/healthz.png`. Sau deploy, nên bổ sung screenshot Render Dashboard và domain `/healthz`.
