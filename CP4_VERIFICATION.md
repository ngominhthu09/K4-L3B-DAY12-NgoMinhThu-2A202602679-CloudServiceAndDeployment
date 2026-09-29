# Tự kiểm chứng CP4

Các lệnh dưới đây chạy tại thư mục gốc repo. Chọn đúng khối lệnh cho
PowerShell hoặc Bash/WSL. Không cần nhập hoặc hiển thị API key: lệnh đọc key
từ `.env` cũng đặt biến môi trường cho Docker Compose dùng cùng giá trị.

## 1. Chạy checkpoint (PowerShell)

```powershell
.\.venv\Scripts\python.exe -m pytest tests/test_cp4.py -v
```

Kiểm tra lại các checkpoint trước nếu cần:

```powershell
.\.venv\Scripts\python.exe -m pytest tests/test_cp1.py tests/test_cp2.py tests/test_cp3.py tests/test_cp4.py -v -m "not docker"
```

Trong WSL, dùng `./.venv/Scripts/python.exe` thay cho đường dẫn PowerShell.
Repo hiện dùng môi trường Python Windows; không cần lệnh `python` trong WSL.

## 2. Build bản mới và chạy 3 instance

PowerShell:

```powershell
$env:AGENT_API_KEY = & .\.venv\Scripts\python.exe -c "from dotenv import dotenv_values; print(dotenv_values('.env')['AGENT_API_KEY'], end='')"
docker compose up -d --build --scale agent=3
docker compose ps
```

Bash/WSL:

```bash
export AGENT_API_KEY="$(./.venv/Scripts/python.exe -c 'from dotenv import dotenv_values; print(dotenv_values(".env")["AGENT_API_KEY"], end="")')"
docker compose up -d --build --scale agent=3
docker compose ps
```

Compose cấp mỗi agent một cổng host trong dải **8000–8002**. Cả ba dùng chung
Redis. `docker compose ps` cho biết cổng ứng với từng container. Ba cổng này
cần trống trước khi tạo container mới. Đợi cả ba agent báo `healthy` trước
khi gửi request.

Chỉ gọi `localhost:8000` thì request chỉ đến instance giữ cổng đó. Vòng lặp
bên dưới chủ động luân phiên các cổng để kiểm chứng lịch sử dùng chung,
không cần Nginx cho phần bắt buộc của CP4.

## 3. Kiểm tra health, readiness và lịch sử dùng chung

PowerShell (sau bước 2):

```powershell
curl.exe -i http://localhost:8000/health
curl.exe -i http://localhost:8000/ready

$userId = "cp4-" + [guid]::NewGuid().ToString('N')
$headers = @{ 'X-API-Key' = $env:AGENT_API_KEY; 'X-User-Id' = $userId }
for ($i = 0; $i -lt 5; $i++) {
    $port = 8000 + ($i % 3)
    $body = @{ question = "CP4 turn $i" } | ConvertTo-Json -Compress
    $response = Invoke-RestMethod -Method Post -Uri "http://localhost:$port/ask" -Headers $headers -ContentType 'application/json' -Body $body
    "port=$port history_length=$($response.history_length)"
}
```

Bash/WSL (sau bước 2):

```bash
curl -i http://localhost:8000/health
curl -i http://localhost:8000/ready

user_id="cp4-$(date +%s)-$RANDOM"
for i in $(seq 0 4); do
  port=$((8000 + i % 3))
  printf 'port=%s history_length=' "$port"
  curl -sS "http://localhost:$port/ask" \
    -H "Content-Type: application/json" \
    -H "X-API-Key: ${AGENT_API_KEY}" \
    -H "X-User-Id: $user_id" \
    -d "{\"question\":\"CP4 turn $i\"}" \
    | ./.venv/Scripts/python.exe -c 'import json,sys; print(json.load(sys.stdin)["history_length"])'
done
```

Kỳ vọng: `/health` và `/ready` trả 200; lịch sử lần lượt là **0, 2, 4, 6, 8**
dù đổi cổng. Mỗi lần chạy dùng user mới để không vướng lịch sử hoặc quota
của lần trước. Lịch sử tăng tối đa tới 20 message, sau đó giữ 20 message mới
nhất; TTL là 7 ngày kể từ lần ghi cuối.

## 4. Phân biệt liveness và readiness khi Redis dừng (PowerShell)

Thao tác này tạm ngừng Redis của lab; `finally` khởi động lại Redis.

```powershell
docker compose stop redis
try {
    curl.exe -i http://localhost:8000/health
    curl.exe -i http://localhost:8000/ready
} finally {
    docker compose start redis
}
```

Kỳ vọng: `/health` vẫn 200; `/ready` trả 503 với
`{"status":"not ready","redis":false}`. Sau khi Redis sẵn sàng lại,
`/ready` trở về 200.

## 5. Quan sát shutdown

Chạy trong PowerShell hoặc Bash/WSL:

```text
docker compose stop agent
docker compose logs --tail=80 agent
docker compose up -d --scale agent=3
```

Kỳ vọng trong log: Uvicorn bắt đầu shutdown, có sự kiện `service_stopped` và
hoàn tất dừng ứng dụng. `exec` trong Dockerfile cho Uvicorn nhận tín hiệu
trực tiếp; handler đặt cờ shutdown rồi chuyển tiếp tín hiệu cho Uvicorn.
Uvicorn có tối đa 25 giây xử lý request đang chạy, Docker chờ 30 giây.

Khi cờ shutdown bật, cả hai probe trả 503. Sau khi Uvicorn đóng socket,
`curl` có thể báo mất kết nối thay vì nhận 503. Bộ test CP4 kiểm tra riêng
trạng thái cờ và việc chuyển tiếp signal; một lần stop lúc không có request
chưa chứng minh được việc giữ request đang xử lý.

Nếu có lỗi, lấy traceback thật từ `docker compose logs --tail=100 agent`.
Các kết quả trên là kỳ vọng để đối chiếu, không phải kết quả đã chạy.
