# Thông Tin Deploy — Checkpoint 5

Tài liệu này đã chuẩn bị sẵn cho bước deploy thủ công. Chưa có thao tác
cloud/GitHub credential nào được thực hiện trong worktree này.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Vũ Văn Phong |
| Mã học viên | 2A202601647 |
| Repo name dự kiến | `DAY12-2A202601647-VuVanPhong` |
| Repo | https://github.com/VuVanPhong123/DAY12-2A202601647-VuVanPhong |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-production-f780.up.railway.app/ |
| Platform | Railway |
| Ngày deploy | 2026-08-10 |

## Biến Môi Trường Cần Set Trên Cloud

Chỉ nhập giá trị ở dashboard của platform; không ghi secret vào repo.

| Biến | Trạng thái | Nguồn/ghi chú |
|------|------------|---------------|
| `PORT` | Platform tự gán | Không ghi đè bằng secret; Dockerfile đọc `$PORT` |
| `AGENT_API_KEY` | Đã set trên Railway | Secret đặt trong dashboard |
| `REDIS_URL` |  Đã set bằng Redis service Railway | Redis add-on hoặc Redis managed/Upstash |
| `RATE_LIMIT_PER_MINUTE` | 10 | `10` |
| `MONTHLY_BUDGET_USD` | 10.0 | `10.0` |
| `LOG_LEVEL` | INFO | `INFO` |

## Lệnh Smoke Test Sau Deploy

Thay marker `PENDING_MANUAL_PUBLIC_URL` bằng URL thật sau khi platform cấp
domain. Không dán API key vào file này.

```bash
BASE_URL="PENDING_MANUAL_PUBLIC_URL"

# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i "$BASE_URL/health"

# 2. Readiness — mong đợi 200 {"status":"ready"}
curl -i "$BASE_URL/ready"

# 3. Không có API key — mong đợi 401
curl -i -X POST "$BASE_URL/ask" \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — lấy từ biến môi trường local, không ghi vào repo
curl -i -X POST "$BASE_URL/ask" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối dự kiến trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST "$BASE_URL/ask" \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

![Văn bản thay thế](evidence_assets\dashboard.png)
![Văn bản thay thế](evidence_assets\health.png)

## Ảnh Chụp Màn Hình

Sau khi deploy thủ công, bổ sung:

- `screenshots/dashboard.png` — dashboard service thật.
- `screenshots/health.png` — kết quả gọi `/health` thật.

Trạng thái hiện tại: `PENDING_MANUAL_SCREENSHOT`.

## Ghi Chú

Không bật `LOCAL_FALLBACK=true` để giả lập CP5. CP5 và bằng chứng online sẽ
được hoàn thiện sau khi user tạo service, nhập secret/Redis trên cloud và lấy
URL thật.
