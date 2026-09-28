# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Phạm Quang Đạt |
| Mã học viên | 2A202602704 |
| Repo | https://github.com/datpq-alpha/K4-L3A-DAY12-PhamQuangDat-2A202602704-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-jmni.onrender.com |
| Platform |  Render |
| Ngày deploy | 28/09/2026 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Render Key Value, connection string được Render tự động gắn |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i <URL>/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i <URL>/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST <URL>/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```
#1: curl.exe -i "https://day12-agent-jmni.onrender.com/health"

HTTP/1.1 200 OK
Date: Mon, 28 Sep 2026 10:53:59 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
cf-cache-status: DYNAMIC
rndr-id: f8a8395d-5ce6-4ce7
Server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
CF-RAY: a4223498dc12fd0a-SIN
alt-svc: h3=":443"; ma=86400

{"status":"ok","service":"day12-agent","version":"1.0.0"}

#2: curl.exe -i "https://day12-agent-jmni.onrender.com/ready" 

HTTP/1.1 200 OK
Date: Mon, 28 Sep 2026 10:55:57 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
cf-cache-status: DYNAMIC
rndr-id: 034d930a-b744-4f21
Server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
CF-RAY: a42237fed9a3f9e6-SIN
alt-svc: h3=":443"; ma=86400

{"status":"ready","redis":true}

#3: curl.exe -i -X POST "https://day12-agent-jmni.onrender.com/ask" -H "Content-Type: application/json" --data-raw '{\"question\":\"Hello\"}'

HTTP/1.1 401 Unauthorized
Date: Mon, 28 Sep 2026 10:58:49 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
cf-cache-status: DYNAMIC
rndr-id: 01ea16b2-33c0-498e
Server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
CF-RAY: a4223c3479d040e8-SIN
alt-svc: h3=":443"; ma=86400

{"detail":"invalid or missing API key"}

#4: $env:PYTHONIOENCODING="utf-8"; .\.venv\Scripts\python.exe -c "import json,uuid,httpx; from dotenv import dotenv_values; c=dotenv_values('.env'); r=httpx.post('https://day12-agent-jmni.onrender.com/ask',headers={'X-API-Key':c['DEPLOY_API_KEY'],'X-User-Id':'deploy-doc-'+uuid.uuid4().hex},json={'question':'Deploy là gì?'},timeout=60); print('STATUS:',r.status_code); print(json.dumps(r.json(),ensure_ascii=False,indent=2))"

STATUS: 200
{
  "answer": "Câu hỏi hay. Deploy là gì thường được giải quyết bằng cách chuẩn hóa môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud.",
  "user_id": "deploy-doc-d44e0239698344ab82bf19fe38c62a12",
  "history_length": 0,
  "cost_usd": 2.145e-05,
  "tokens": {
    "in": 3,
    "out": 35
  }
}

#5: $k=((Get-Content .env | Where-Object { $_ -match '^DEPLOY_API_KEY=' } | Select-Object -First 1) -replace '^DEPLOY_API_KEY=','').Trim(); $uid="rate-test-$([DateTimeOffset]::UtcNow.ToUnixTimeSeconds())"; 1..15 | ForEach-Object { curl.exe -s -o NUL -w "%{http_code} " -X POST "https://day12-agent-jmni.onrender.com/ask" -H "Content-Type: application/json" -H "X-API-Key: $k" -H "X-User-Id: $uid" --data-raw '{\"question\":\"test\"}' }; Write-Host

200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl


