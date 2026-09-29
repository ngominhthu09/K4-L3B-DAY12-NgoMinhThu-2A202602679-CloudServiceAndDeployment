# Tự kiểm chứng CP5 trên Railway

Service đã deploy tại:

```text
https://day12-agent-production-f5fb.up.railway.app
```

Chạy các lệnh sau trong một terminal. Không dán API key vào terminal, ảnh chụp
màn hình hoặc tài liệu nộp bài.

## PowerShell

```powershell
$baseUrl = 'https://day12-agent-production-f5fb.up.railway.app'

# Liveness: mong đợi HTTP 200 và status = ok.
curl.exe -i "$baseUrl/health"

# Readiness: mong đợi HTTP 200 và redis = true.
curl.exe -i "$baseUrl/ready"

# Không gửi key: mong đợi HTTP 401.
curl.exe -i -X POST "$baseUrl/ask" -H "Content-Type: application/json" -d '{"question":"Hello"}'

# Đọc key cục bộ, không in key; gọi có xác thực: mong đợi HTTP 200.
$env:AGENT_API_KEY = & .\.venv\Scripts\python.exe -c "from dotenv import dotenv_values; print(dotenv_values('.env')['AGENT_API_KEY'], end='')"
curl.exe -i -X POST "$baseUrl/ask" -H "Content-Type: application/json" -H "X-API-Key: $env:AGENT_API_KEY" -H "X-User-Id: cp5-test" -d '{"question":"Deploy là gì?"}'

# Test CP5 của lab.
.\.venv\Scripts\python.exe -m pytest tests/test_cp5.py -v
```

## Bash / WSL

```bash
base_url='https://day12-agent-production-f5fb.up.railway.app'

curl -i "$base_url/health"
curl -i "$base_url/ready"
curl -i -X POST "$base_url/ask" \
  -H 'Content-Type: application/json' \
  -d '{"question":"Hello"}'

export AGENT_API_KEY="$(./.venv/Scripts/python.exe -c 'from dotenv import dotenv_values; print(dotenv_values(".env")["AGENT_API_KEY"], end="")')"
curl -i -X POST "$base_url/ask" \
  -H 'Content-Type: application/json' \
  -H "X-API-Key: ${AGENT_API_KEY}" \
  -H 'X-User-Id: cp5-test' \
  -d '{"question":"Deploy là gì?"}'

./.venv/Scripts/python.exe -m pytest tests/test_cp5.py -v
```

Nếu kiểm tra endpoint có key trả 401, lấy lại key trong `.env` bằng khối lệnh
trên; không gõ tay key. Nếu `/ready` trả 503, xem Railway dashboard →
`day12-agent` → Variables để bảo đảm `REDIS_URL` đang tham chiếu Redis Railway.

Sau khi các lệnh xanh, chụp Railway dashboard vào `screenshots/dashboard.png`
và kết quả `/health` vào `screenshots/health.png`. Sau đó thay phần “Kết Quả
Chạy Thật” trong `DEPLOYMENT.md` bằng output thật của bạn.
