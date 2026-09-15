# Session 5 — Bash & Checkpoint

## Mục tiêu buổi học

- Viết smoke test script kiểm tra HTTP status, version, timeout và exit code.
- Hiểu vì sao CI/CD dựa vào exit code để quyết định PASS/FAIL.
- Dùng `curl -fsS` để biến HTTP 4xx/5xx thành failure cho automation.
- Hiểu `set -Eeuo pipefail`, quoting và pipeline failure.
- Gây lỗi API có kiểm soát, đọc evidence và khoanh vùng giữa Nginx và Node API.
- Viết runbook ngắn theo flow: triệu chứng → giả thuyết → kiểm tra → cách sửa.

Mental model xuyên suốt:

```text
Smoke test
   ↓
HTTP request
   ↓
exit code
   ↓
0       → PASS
!= 0    → FAIL
   ↓
CI/CD quyết định có được tiếp tục release hay không
```

Nguyên tắc chính:

> Automation không chỉ cần in ra lỗi. Nó phải trả đúng exit code để hệ thống phía ngoài có thể ra quyết định.

---

## 1. Exit code trong Linux và CI/CD

Sau mỗi command, Bash lưu exit code của command gần nhất trong:

```bash
echo $?
```

Quy ước:

```text
0       = success
!= 0    = failure
```

Ví dụ:

```bash
ls /etc/nginx
echo $?
```

Nếu command thành công:

```text
0
```

Nếu command lỗi:

```bash
ls /folder-does-not-exist
echo $?
```

có thể nhận:

```text
2
```

Trong CI/CD:

```text
script exit 0
      ↓
Jenkins stage PASS

script exit != 0
      ↓
Jenkins stage FAIL
      ↓
không deploy tiếp
```

Vì vậy đoạn này chưa đủ:

```bash
echo "FAIL"
```

Nếu script vẫn kết thúc với exit code `0`, Jenkins có thể vẫn coi step là thành công.

Phải trả failure thực sự:

```bash
echo "FAIL"
exit 1
```

---

## 2. `curl` và HTTP failure

### `curl -sS`

```bash
curl -sS http://localhost/not-found
```

Ý nghĩa:

```text
-s = silent, ẩn progress bar
-S = vẫn hiện error khi có lỗi curl
```

Nhưng HTTP `404` hoặc `500` vẫn có thể không làm `curl` fail theo exit code.

### `curl -fsS`

```bash
curl -fsS http://localhost/not-found
```

`-f` nghĩa là:

```text
HTTP >= 400 → curl fail
```

Ví dụ:

```bash
curl -fsS http://localhost/not-found
echo $?
```

Có thể nhận:

```text
curl: (22) The requested URL returned error: 404
22
```

Mental model:

```text
-sS
→ HTTP response có thể là 4xx/5xx nhưng command chưa chắc fail

-fsS
→ HTTP 4xx/5xx được chuyển thành non-zero exit code
→ phù hợp cho automation / CI/CD
```

---

## 3. Hai curl exit code quan trọng trong lab

### Exit code 22

Ví dụ:

```bash
curl -fsS http://localhost/api/healthz
```

trả:

```text
curl: (22) ... 502
```

Điều này cho biết:

```text
đã kết nối được tới HTTP server
        ↓
đã nhận HTTP response
        ↓
response là 4xx/5xx
```

Nó không phải lỗi TCP connect tới Nginx.

### Exit code 7

Ví dụ:

```bash
curl -fsS http://localhost:3000/healthz
```

trả:

```text
curl: (7) Failed to connect
```

Điều này cho biết:

```text
TCP connection tới host:port không thiết lập được
```

Trong lab, nếu:

```text
/api/healthz qua Nginx → 502 / curl 22
localhost:3000/healthz → curl 7
```

thì nghi phạm chính là:

```text
Node process
systemd service
listener port 3000
```

---

## 4. Timeout cho smoke test

Service không nhất thiết phải crash hoàn toàn.

Có thể xảy ra:

```text
process còn sống
port còn LISTEN
TCP connect được
nhưng application treo
```

Vì vậy smoke test nên có timeout.

Ví dụ:

```bash
curl \
  --connect-timeout 2 \
  --max-time 5 \
  http://localhost/api/healthz
```

Ý nghĩa:

```text
--connect-timeout 2
→ tối đa 2 giây để thiết lập connection

--max-time 5
→ toàn bộ request tối đa 5 giây
```

Mental model:

```text
DNS
 ↓
TCP connect     ← connect-timeout
 ↓
HTTP request
 ↓
application xử lý
 ↓
response
 └────────────── max-time ──────────────┘
```

---

## 5. Bash strict mode

Script automation thường bắt đầu bằng:

```bash
#!/usr/bin/env bash

set -Eeuo pipefail
```

### `set -e`

Dừng script khi command trả non-zero trong các context mà Bash áp dụng `errexit`.

Ví dụ:

```bash
set -e

echo "step 1"
ls /folder-does-not-exist
echo "step 2"
```

`step 2` sẽ không chạy.

Ứng dụng CI/CD:

```text
npm test FAIL
      ↓
script STOP
      ↓
không deploy artifact lỗi
```

### `set -u`

Fail khi dùng biến chưa được định nghĩa.

Ví dụ:

```bash
set -u

echo "$API_URL"
```

Nếu `API_URL` chưa tồn tại thì script fail thay vì âm thầm dùng giá trị rỗng.

### `set -o pipefail`

Mặc định, exit status của pipeline thường lấy từ command cuối.

Ví dụ:

```bash
false | echo "hello"
echo $?
```

Có thể nhận:

```text
hello
0
```

Dù command `false` đã fail.

Bật:

```bash
set -o pipefail
```

thì pipeline sẽ fail nếu một command bên trong pipeline fail.

### `-E`

Giúp `ERR` trap được kế thừa tốt hơn trong function/subshell.

Trong buổi này chỉ cần nhớ:

```text
-e        command lỗi → dừng script
-u        biến undefined → fail
pipefail  lỗi bên trong pipeline → pipeline fail
-E        ERR trap được kế thừa tốt hơn
```

`set -Eeuo pipefail` hữu ích nhưng không phải cơ chế bắt mọi loại bug Bash.

---

## 6. Quoting

Ví dụ:

```bash
FILE="my release.txt"
```

Không quote:

```bash
rm $FILE
```

Shell có thể word-split thành:

```text
my
release.txt
```

Quote đúng:

```bash
rm "$FILE"
```

Rule thực tế:

> Mặc định quote biến Bash bằng `"${VAR}"`, trừ khi thật sự muốn word splitting hoặc globbing.

Ví dụ:

```bash
curl "${BASE_URL}/api/healthz"
```

---

## 7. Smoke test script

Ví dụ script đã dùng trong lab:

```bash
#!/usr/bin/env bash

set -Eeuo pipefail

BASE_URL="${BASE_URL:-http://127.0.0.1}"

HEALTH_URL="${BASE_URL}/api/healthz"
VERSION_URL="${BASE_URL}/api/version"

echo "Smoke test target: ${BASE_URL}"

echo "[1/2] Checking health..."

if ! curl \
  -fsS \
  --connect-timeout 2 \
  --max-time 5 \
  "${HEALTH_URL}" \
  > /dev/null
then
  echo "FAIL: health endpoint is unavailable"
  exit 1
fi

echo "PASS: health endpoint"

echo "[2/2] Checking version..."

VERSION="$(
  curl \
    -fsS \
    --connect-timeout 2 \
    --max-time 5 \
    "${VERSION_URL}"
)"

if [[ -z "${VERSION}" ]]; then
  echo "FAIL: version is empty"
  exit 1
fi

echo "PASS: version=${VERSION}"
echo "Smoke test PASSED"
```

Cấp quyền execute:

```bash
chmod +x ~/smoke-test.sh
```

Chạy:

```bash
~/smoke-test.sh
echo $?
```

Kỳ vọng app khỏe:

```text
Smoke test PASSED
0
```

Kỳ vọng app lỗi:

```text
FAIL: ...
non-zero exit code
```

---

## 8. Incident lab — Node API chết

Dừng service:

```bash
sudo systemctl stop myapp
```

Kiểm tra process:

```bash
pgrep -af node
```

Kiểm tra listener:

```bash
ss -ltnp | grep 3000
```

Nếu API đã chết thì không còn listener phù hợp trên `:3000`.

Kiểm tra trực tiếp upstream:

```bash
curl -fsS http://127.0.0.1:3000/healthz
echo $?
```

Kỳ vọng:

```text
curl: (7) Failed to connect
7
```

Kiểm tra qua Nginx:

```bash
curl -fsS http://127.0.0.1/api/healthz
echo $?
```

Có thể nhận:

```text
curl: (22) The requested URL returned error: 502
22
```

Suy luận:

```text
Nginx reachable
     ↓
HTTP 502
     ↓
Nginx không gọi được upstream
     ↓
check :3000
     ↓
TCP connection failed
     ↓
check Node process / systemd
```

---

## 9. Dùng evidence để chứng minh root cause

Không nên thấy `502` rồi restart mọi thứ ngay.

Thu thập evidence trước.

### systemd

```bash
sudo systemctl status myapp
```

### Journal

```bash
sudo journalctl -u myapp -n 50 --no-pager
```

Hoặc:

```bash
sudo journalctl -u myapp --since "10 minutes ago"
```

### Nginx error log

```bash
sudo tail -n 50 /var/log/nginx/error.log
```

Có thể thấy:

```text
connect() failed (111: Connection refused) while connecting to upstream
```

Đây là evidence rằng:

```text
Nginx đang hoạt động
Nginx đã cố connect upstream
upstream từ chối connection
```

Debug flow tốt:

```text
symptom
   ↓
collect evidence
   ↓
localize layer
   ↓
identify root cause
   ↓
fix
   ↓
verify
```

Không nên debug theo kiểu:

```text
symptom
   ↓
restart everything
```

---

## 10. Phục hồi service và verify

Khởi động lại:

```bash
sudo systemctl start myapp
```

Verify từng tầng:

```bash
sudo systemctl status myapp
```

```bash
ss -ltnp | grep 3000
```

```bash
curl -fsS http://127.0.0.1:3000/healthz
```

```bash
curl -fsS http://127.0.0.1/api/healthz
```

Cuối cùng:

```bash
~/smoke-test.sh
echo $?
```

Kỳ vọng:

```text
PASS
0
```

---

## 11. Khi Node OK nhưng Nginx trả 502

Case:

```bash
ss -ltnp | grep 3000
```

có:

```text
LISTEN ... 127.0.0.1:3000
```

Và:

```bash
curl -fsS http://127.0.0.1:3000/healthz
```

trả thành công.

Nhưng:

```bash
curl -fsS http://127.0.0.1/api/healthz
```

vẫn `502`.

Lúc này Node không còn là nghi phạm chính.

Ưu tiên kiểm tra Nginx:

```bash
sudo nginx -t
```

Sau đó xem config Nginx thực sự đang load:

```bash
sudo nginx -T
```

Tìm `proxy_pass`:

```bash
sudo nginx -T | grep -n "proxy_pass"
```

Hoặc:

```bash
sudo grep -R "proxy_pass" /etc/nginx/
```

Kiểm tra error log:

```bash
sudo tail -n 50 /var/log/nginx/error.log
```

Một config sai port nhưng syntax vẫn hợp lệ:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:3001;
}
```

trong khi Node chạy `:3000`.

Điểm quan trọng:

```text
nginx -t successful
!=
routing đúng
```

Sau khi sửa config:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Ưu tiên `reload` khi chỉ thay đổi config hợp lệ thay vì restart không cần thiết.

---

## 12. `proxy_pass` và trailing slash

Đây là lỗi routing rất dễ gặp.

### Có slash cuối

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:3000/;
}
```

Request:

```text
/api/healthz
```

được gửi xuống upstream thành:

```text
/healthz
```

### Không có slash cuối

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:3000;
}
```

Request:

```text
/api/healthz
```

được giữ path:

```text
/api/healthz
```

Nếu Node chỉ có:

```text
GET /healthz
```

thì backend có thể trả `404`.

Mental model:

```text
502
→ thường nghĩ tới upstream connectivity / protocol / process / port

404 từ backend
→ request đã đi tới application
→ ưu tiên kiểm tra path / routing
```

---

## 13. Pipeline failure với `tee`

Một case CI/CD quan trọng:

```bash
npm test | tee test.log
```

Nếu:

```text
npm test → exit 1
tee      → exit 0
```

mà shell không bật `pipefail`, pipeline có thể bị coi là:

```text
PASS
```

Flow nguy hiểm:

```text
npm test FAIL
      ↓
tee PASS
      ↓
pipeline exit 0
      ↓
Jenkins PASS
      ↓
có thể tiếp tục deploy
```

Bật:

```bash
set -o pipefail
```

để failure bên trong pipeline được propagate.

---

## 14. `pipefail` phải nằm ở shell tạo pipeline

Case:

```bash
./smoke-test.sh | tee smoke.log
```

Bên trong `smoke-test.sh` có:

```bash
exit 1
```

nhưng Jenkins vẫn success.

Root cause có thể là shell bên ngoài chưa bật `pipefail`.

Điểm quan trọng:

> `set -o pipefail` phải được bật trong shell đang thực thi pipeline có ký tự `|`.

Ví dụ đúng:

```bash
set -o pipefail
./smoke-test.sh | tee smoke.log
```

Hoặc:

```bash
bash -o pipefail -c './smoke-test.sh | tee smoke.log'
```

Mental model:

```text
smoke-test.sh = 1
tee           = 0
        ↓
pipefail
        ↓
pipeline = non-zero
        ↓
Jenkins = FAIL
```

Chỉ đặt `set -o pipefail` bên trong `smoke-test.sh` không tự động thay đổi pipeline do shell cha tạo ra.

---

## 15. Runbook incident mẫu

### Triệu chứng

```text
GET /api/healthz trả 502
```

### Giả thuyết

```text
Nginx hoạt động nhưng Node upstream :3000 không chạy
```

### Kiểm tra

```text
1. curl /api/healthz
   → HTTP 502

2. curl localhost:3000/healthz
   → connection failed

3. ss -ltnp | grep 3000
   → không có listener

4. systemctl status myapp
   → service inactive / failed

5. journalctl -u myapp
   → tìm nguyên nhân process dừng
```

### Cách sửa

```text
1. Xác định và sửa nguyên nhân Node service lỗi
2. systemctl start/restart myapp
3. Xác nhận :3000 LISTEN
4. Kiểm tra /healthz trực tiếp
5. Kiểm tra /api/healthz qua Nginx
6. Chạy smoke-test.sh
7. Xác nhận exit code = 0
```

---

## 16. Checkpoint kiến thức đã đạt

### Exit code

```text
0       → success
!= 0    → failure
```

### curl

```text
-f → HTTP 4xx/5xx thành failure
-s → silent
-S → vẫn show error
```

### curl exit code

```text
22 → nhận HTTP 4xx/5xx khi dùng -f
7  → TCP connection tới host:port không thiết lập được
```

### Bash strict mode

```text
-e        fail sớm khi command lỗi
-u        fail khi biến undefined
pipefail  propagate lỗi trong pipeline
-E        hỗ trợ ERR trap inheritance
```

### Debug

```text
502 qua Nginx + port upstream không connect được
→ kiểm tra Node process / systemd / port

Node trực tiếp OK nhưng Nginx 502
→ kiểm tra Nginx config / proxy_pass / protocol / log

404 từ backend
→ ưu tiên kiểm tra path / routing
```

### proxy_pass

```text
proxy_pass http://127.0.0.1:3000/;
/api/healthz → /healthz

proxy_pass http://127.0.0.1:3000;
/api/healthz → /api/healthz
```

### CI pipeline

```text
npm test | tee test.log
```

cần `pipefail` ở shell tạo pipeline để test failure không bị `tee` che mất.

---

## 17. Checkpoint tuần 1

Sau Session 1–5 cần có khả năng tự suy luận theo flow:

```text
DNS
 ↓
Route / network path
 ↓
TCP
 ↓
Port / listener
 ↓
TLS
 ↓
HTTP
 ↓
Nginx reverse proxy
 ↓
Node application
```

Khi có lỗi:

```text
1. Quan sát symptom
2. Lấy evidence
3. Xác nhận layer nào đã OK
4. Khoanh vùng layer đầu tiên fail
5. Kiểm tra root cause
6. Sửa
7. Verify lại từ upstream trực tiếp tới đường đi người dùng
8. Chạy smoke test
```

Checkpoint cuối tuần 1:

- Tự khoanh vùng được DNS sai, port sai và process chết.
- Phân biệt `connection refused`, timeout, HTTP 4xx/5xx và routing error.
- Hiểu Nginx reverse proxy và upstream path.
- Viết được smoke test trả đúng exit code.
- Script fail khi API chết và pass sau khi service phục hồi.
- Không restart ngẫu nhiên trước khi thu thập evidence.

---

## Command reference nhanh

```bash
# Exit code command trước
echo $?

# Health check phù hợp automation
curl -fsS --connect-timeout 2 --max-time 5 http://127.0.0.1/api/healthz

# Kiểm tra upstream trực tiếp
curl -fsS http://127.0.0.1:3000/healthz

# Process Node
pgrep -af node

# Listener
ss -ltnp | grep 3000

# systemd
sudo systemctl status myapp
sudo systemctl stop myapp
sudo systemctl start myapp
sudo systemctl restart myapp

# Logs
sudo journalctl -u myapp -n 50 --no-pager
sudo journalctl -u myapp --since "10 minutes ago"

# Nginx
sudo nginx -t
sudo nginx -T
sudo nginx -T | grep -n "proxy_pass"
sudo tail -n 50 /var/log/nginx/error.log
sudo systemctl reload nginx

# Bash pipeline safety
set -o pipefail

# CI example
set -o pipefail
./smoke-test.sh | tee smoke.log
```

---

## Kết luận

Buổi 5 nối toàn bộ kiến thức tuần 1 thành khả năng vận hành thực tế:

```text
không chỉ biết application có lỗi
          ↓
biết lấy evidence
          ↓
biết layer nào fail
          ↓
biết script hóa kiểm tra
          ↓
trả đúng exit code
          ↓
CI/CD có thể tự động chặn release lỗi
```

Bước tiếp theo theo plan là **Session 6 — Docker image có thể dựng lại**, tập trung vào Dockerfile multi-stage, lockfile, build context, `.dockerignore`, base image version và reproducible build.