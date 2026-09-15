# Week 1 Master Summary — Sessions 1 to 5

Tài liệu này tổng hợp toàn bộ kiến thức chính đã học trong 5 buổi đầu của lộ trình DevOps.

Mục tiêu của tuần 1 không phải học thật nhiều command riêng lẻ, mà xây được một mental model thống nhất để trả lời ba câu hỏi:

```text
1. Request đang đi qua những layer nào?
2. Layer nào đã có evidence xác nhận là OK?
3. Layer đầu tiên bị fail nằm ở đâu và root cause thật là gì?
```

Mental model tổng quát:

```text
Developer
   ↓
Git
   ↓
Jenkins Agent
   ↓
Install / Test / Build
   ↓
Artifact
   ↓
Deploy

User Request
   ↓
DNS
   ↓
IP / Route
   ↓
TCP
   ↓
Port / Listener
   ↓
TLS
   ↓
HTTP
   ↓
Nginx
   ↓
Node Application
   ↓
Dependencies
```

Nguyên tắc xuyên suốt:

> Evidence trước, thay đổi sau. Layer nào đã được chứng minh hoạt động thì không quay lại sửa ngẫu nhiên layer đó.

---

# Session 1 — Web Flow, Jenkins Basics & Ubuntu Environment

## 1. Ubuntu VM là môi trường lab

Các command nền tảng:

```bash
whoami
id
pwd
uname -a
ip a
```

Update package:

```bash
sudo apt update && sudo apt upgrade -y
```

Mục tiêu là có một Linux VM độc lập để thực hành process, service, network, Nginx, Docker và CI/CD mà không phụ thuộc máy host.

---

## 2. Jenkins pipeline cơ bản

Flow tổng quát:

```text
Git Repository
    ↓
Checkout
    ↓
Install Dependencies
    ↓
Test
    ↓
Build
    ↓
Artifact
    ↓
Approval
    ↓
Deploy
```

### `agent`

`agent` là nơi Jenkins thực thi command của pipeline.

```text
Jenkins Controller
       ↓ giao job
Jenkins Agent
       ↓
npm ci
npm test
npm run build
```

### `checkout scm`

Lấy source code từ SCM về workspace của Jenkins Agent.

```groovy
checkout scm
```

### `npm ci`

Trong CI thường ưu tiên:

```bash
npm ci
```

thay cho:

```bash
npm install
```

vì `npm ci` dựa trên lockfile và phù hợp hơn với reproducible build.

### Test và exit code

```text
exit 0      → success
exit != 0   → failure
```

Ví dụ:

```text
npm test
   ↓
exit 1
   ↓
Jenkins stage FAIL
   ↓
không nên deploy tiếp
```

### Build artifact

Với React/Vite:

```bash
npm run build
```

thường tạo:

```text
dist/
```

`dist/` là artifact để deploy, không phải source code gốc.

---

## 3. Credentials và secrets

Không hard-code secret trong source hoặc Jenkinsfile.

Sai:

```text
API_KEY="secret"
```

Mental model đúng:

```text
Secret Store
   ↓
CI/CD hoặc Runtime
   ↓
Inject khi cần
```

Nguyên tắc:

```text
Secret không commit vào Git.
Secret chỉ xuất hiện ở nơi thật sự cần dùng.
```

---

## 4. Manual approval

Một release flow có thể có gate:

```text
Test ✅
Build ✅
Artifact ✅
    ↓
Human Approval
    ↓
Production
```

Approval không đảm bảo release chắc chắn không lỗi. Nó chỉ thêm một control point trước hành động rủi ro cao.

Sau này vẫn cần:

```text
health check
monitoring
rollback
```

---

## 5. Browser request flow

Khi browser gọi:

```text
https://app.example.com
```

flow cơ bản:

```text
Browser
   ↓
DNS
   ↓
Destination IP
   ↓
TCP Connection
   ↓
TLS Handshake
   ↓
HTTP Request
   ↓
Server / Application
   ↓
HTTP Response
```

Ghi nhớ:

```text
DNS → TCP/TLS → HTTP → Application
```

Port `443` là default port của HTTPS, không phải port duy nhất có thể chạy HTTPS.

---

## 6. Nginx và application port

Pattern phổ biến:

```text
Internet
   ↓ HTTPS :443
Nginx
   ↓ internal HTTP
Node :3000
```

Node không cần expose `3000` trực tiếp ra Internet nếu Nginx là entry point.

---

## 7. Docker networking concepts đã làm quen

Port mapping:

```text
8080:3000
^^^^ ^^^^
host container
```

Nếu container khác cần gọi Node, application thường phải listen trên:

```text
0.0.0.0:3000
```

chứ không phải chỉ:

```text
127.0.0.1:3000
```

Hai container cùng Compose network có thể giao tiếp qua service name:

```text
http://api:3000
```

mà không cần publish port API ra host.

---

# Session 2 — Linux for Web Applications, systemd & Debugging

## 1. Linux user và process identity

Process chạy dưới user nào thì bị giới hạn bởi permission của user đó.

Ví dụ:

```bash
whoami
```

Nếu chạy Node bằng user `appuser`, process Node chỉ có quyền mà `appuser` có.

Đây là nguyên tắc:

```text
least privilege
```

Không chạy app bằng `root` nếu app không cần quyền root.

Nếu có RCE:

```text
App chạy root
→ attacker có phạm vi quyền rất lớn

App chạy appuser
→ impact bị giới hạn theo permission appuser
```

Concept này sẽ gặp lại ở:

```text
Linux user
AWS IAM
Docker USER
Kubernetes ServiceAccount
Jenkins credentials
```

---

## 2. File permission

Permission format:

```text
-rw-r-----
 │  │  │
 │  │  └── other
 │  └───── group
 └──────── owner
```

Giá trị:

```text
r = 4
w = 2
x = 1
```

Ví dụ:

```text
640 = rw-r-----
```

Nếu file là:

```text
-rw-r----- root appuser config.json
```

thì:

```text
root    → read + write
appuser → read only
others  → no permission
```

Một fix theo least privilege:

```bash
sudo chown root:appuser /opt/myapp/config.json
sudo chmod 640 /opt/myapp/config.json
```

Không dùng `chmod 777` như default fix.

---

## 3. Process và listener

Tìm Node process:

```bash
pgrep -af node
```

Kiểm tra TCP listener:

```bash
ss -ltnp
ss -ltnp | grep :3000
```

Mental model:

```text
Source file
   ↓
Node runtime
   ↓
Process / PID
   ↓
TCP socket
   ↓
:3000
```

Source code tự nó không giữ port. Process đang chạy mới giữ listener.

---

## 4. `127.0.0.1` và `0.0.0.0`

```text
127.0.0.1:3000
→ chỉ loopback của chính machine/container

0.0.0.0:3000
→ listen trên mọi IPv4 interface
```

Nếu:

```text
curl localhost:3000 → success
curl VM-IP:3000     → fail
```

thì chưa nên kết luận app chết. Cần kiểm tra:

```text
bind address
firewall
routing
security rules
```

---

## 5. systemd

Unit file ví dụ:

```ini
[Service]
User=appuser
Group=appuser
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/node /opt/myapp/server.js
Environment=PORT=3000
Restart=on-failure
RestartSec=3
```

Command quan trọng:

```bash
sudo systemctl start myapp
sudo systemctl stop myapp
sudo systemctl restart myapp
sudo systemctl enable myapp
sudo systemctl enable --now myapp
```

Phân biệt:

```text
start
→ chạy service ngay

enable
→ cấu hình để service tự start khi boot
```

`enabled` không đồng nghĩa:

```text
running
healthy
```

---

## 6. `daemon-reload`

Sau khi sửa unit file:

```bash
sudo systemctl daemon-reload
sudo systemctl restart myapp
```

Mental model:

```text
edit unit file
   ↓
daemon-reload
   ↓
systemd đọc definition mới
   ↓
restart
   ↓
process mới dùng config mới
```

`daemon-reload` không tự restart application.

---

## 7. Logs với `journalctl`

```bash
sudo journalctl -u myapp
sudo journalctl -u myapp -n 50 --no-pager
sudo journalctl -u myapp -f
sudo journalctl -u myapp -b
```

Mental model:

```text
Node stdout/stderr
      ↓
systemd journal
      ↓
journalctl
```

---

## 8. Incident EACCES

Symptom:

```text
Connection refused
```

Root cause chain thực tế:

```text
Connection refused
   ↓
không có listener :3000
   ↓
Node process chết
   ↓
startup failed
   ↓
EACCES config.json
   ↓
permission / ownership sai
```

Verify bằng đúng identity của service:

```bash
sudo -u appuser cat /opt/myapp/config.json
```

Nguyên tắc:

```text
Observation
   ↓
Hypothesis
   ↓
Verification
   ↓
Fix
   ↓
Verify again
```

---

## 9. Process alive không đồng nghĩa app healthy

Có thể xảy ra:

```text
systemctl status → active
process          → tồn tại
:3000            → LISTEN
/healthz         → HTTP 500
```

Kết luận:

```text
process health ✅
application health ❌
```

Đây là nền tảng để hiểu Kubernetes probes sau này.

---

# Session 3 — Networking for Debugging

Mental model chính:

```text
Client
  ↓
DNS
  ↓
Destination IP
  ↓
Route
  ↓
TCP
  ↓
Port / Listener
  ↓
TLS
  ↓
HTTP
  ↓
Reverse Proxy / Application
```

---

## 1. IP, subnet và route

Ví dụ:

```text
10.0.1.20/24
```

network:

```text
10.0.1.0/24
```

Route example:

```text
default via 10.0.1.1 dev eth0
10.0.1.0/24 dev eth0
```

Nếu destination cùng subnet, packet dùng route trực tiếp.

Nếu destination ngoài subnet và không có route cụ thể hơn, Linux dùng default gateway.

Command:

```bash
ip addr
ip route
ip route get <destination-ip>
```

---

## 2. TCP handshake

```text
Client              Server
SYN  ──────────────►
     ◄──────── SYN-ACK
ACK  ──────────────►
TCP established
```

TLS chỉ bắt đầu sau khi TCP connection được thiết lập.

---

## 3. Connection refused

Thường biểu hiện:

```text
SYN
 ↓
RST
```

Mental model:

```text
network path có phản hồi
        ↓
TCP connection bị từ chối
        ↓
ưu tiên listener / process / port
```

Kiểm tra:

```bash
ss -ltnp
systemctl status <service>
journalctl -u <service>
```

`Connection refused` không có nghĩa TCP đã connect thành công.

---

## 4. Timeout

Timeout thường khiến ta nghi:

```text
routing
firewall
security group / ACL
packet filtering
network path
```

Với timeout, chưa thể khẳng định route hoặc TCP path là OK.

---

## 5. DNS

Nếu:

```text
Could not resolve host
```

thì:

```text
DNS   = ERROR
TCP   = chưa tới
TLS   = chưa tới
HTTP  = chưa tới
```

Command:

```bash
dig example.com
dig +short example.com
getent hosts example.com
```

`getent` hữu ích khi debug `/etc/hosts`.

DNS resolve được nhưng trả sai IP thì DNS vẫn có thể là root cause.

Bypass DNS để verify:

```bash
curl -vk --resolve api.example.com:443:10.0.1.50 \
  https://api.example.com/healthz
```

---

## 6. TLS error

Nếu thấy:

```text
Connected to ... port 443
```

rồi mới có certificate error thì:

```text
DNS   = OK
Route = OK
TCP   = OK
443   = OK
TLS   = ERROR
HTTP  = chưa tới
```

Ví dụ:

```text
expired certificate
hostname mismatch
```

---

## 7. HTTP 500

Nếu nhận:

```text
HTTP/1.1 500 Internal Server Error
```

thì client đã đi qua:

```text
DNS
Route
TCP
TLS nếu HTTPS
HTTP
```

Lỗi nằm ở application hoặc dependency phía sau.

---

## 8. HTTP 502

`502 Bad Gateway` thường có nghĩa:

```text
Client → Nginx ✅
Nginx → upstream ❌ hoặc upstream response không hợp lệ
```

Debug:

```bash
curl http://127.0.0.1:3000/healthz
ss -ltnp | grep :3000
systemctl status myapp
journalctl -u myapp -n 50 --no-pager
```

---

## 9. Ba incident quan trọng

### DNS sai

```text
hostname → sai IP
```

Check:

```bash
getent hosts <hostname>
```

### Sai port

```text
Node process sống
:3000 LISTEN
client gọi :3001
```

### Process chết

```text
client gọi đúng :3000
nhưng không còn listener
```

Debug:

```bash
ss -ltnp | grep :3000
pgrep -af node
systemctl status myapp
journalctl -u myapp
```

Điểm quan trọng:

> Cùng một symptom có thể có nhiều root cause khác nhau. Evidence là thứ phân biệt chúng.

---

# Session 4 — Nginx, Reverse Proxy & Browser

## 1. Public URL và internal upstream

Browser chỉ cần biết public entry point:

```text
http://<VM-IP>/api/healthz
```

Nginx có thể proxy nội bộ tới:

```text
http://127.0.0.1:3000
```

Flow:

```text
Browser
   ↓
Nginx :80/:443
   ├── static SPA
   └── /api/*
          ↓
       Node :3000
```

`127.0.0.1` luôn là localhost của machine/process đang sử dụng địa chỉ đó.

---

## 2. Reverse proxy

Config:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:3000/;
}
```

Browser gọi:

```text
/api/healthz
```

Node nhận:

```text
/healthz
```

---

## 3. `proxy_pass` trailing slash

Có slash cuối:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:3000/;
}
```

```text
/api/healthz → /healthz
```

Không có slash cuối:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:3000;
}
```

```text
/api/healthz → /api/healthz
```

Nếu backend chỉ có `/healthz`, case thứ hai có thể trả `404`.

Mental model:

```text
502
→ ưu tiên upstream connectivity / process / port / protocol

404 từ backend
→ request đã tới app
→ ưu tiên route / path
```

---

## 4. Test config trước reload

```bash
sudo nginx -t
sudo systemctl reload nginx
```

`nginx -t` chỉ xác nhận syntax/config có thể parse, không đảm bảo routing logic đúng.

Xem config thực tế Nginx đang load:

```bash
sudo nginx -T
sudo nginx -T | grep -n "proxy_pass"
```

---

## 5. SPA deep route

Với React Router:

```text
/dashboard
```

refresh browser tạo request thật tới Nginx.

Cần:

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

Flow:

```text
GET /dashboard
   ↓
không có file /dashboard
   ↓
/index.html
   ↓
React load
   ↓
React Router xử lý /dashboard
```

Nếu không có fallback, refresh deep route có thể `404`.

---

## 6. Nginx location matching

Với:

```nginx
location / {
    try_files $uri $uri/ /index.html;
}

location /api/ {
    proxy_pass http://127.0.0.1:3000/;
}
```

`/api/healthz` match cả `/` và `/api/`, nhưng `/api/` cụ thể hơn nên được chọn.

Điều này ngăn API request bị SPA fallback trả `index.html`.

---

## 7. CORS không sửa được 502

CORS là browser policy cho cross-origin request.

`502` là server-side failure giữa reverse proxy và upstream.

Nếu:

```bash
curl http://127.0.0.1/api/healthz
```

cũng trả `502`, thì CORS không phải root cause vì `curl` không enforce browser CORS policy.

---

## 8. Same origin

Origin được xác định bởi:

```text
scheme + host + port
```

Path không quyết định origin.

Ví dụ cùng origin:

```text
http://10.0.0.20/
http://10.0.0.20/api/healthz
```

Khác origin nếu khác:

```text
scheme
host
port
```

Reverse proxy `/api/` giúp frontend và API dùng cùng public origin.

---

## 9. Forwarded headers

```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

Quan trọng trong flow:

```text
Browser --HTTPS--> Nginx --HTTP--> Node
```

Node có thể biết protocol gốc nhờ:

```text
X-Forwarded-Proto: https
```

---

## 10. Cookie attributes

```text
Secure
→ chỉ gửi cookie qua HTTPS

HttpOnly
→ JavaScript không đọc được bằng document.cookie

SameSite
→ kiểm soát gửi cookie trong context liên quan site khác
```

CORS và cookie attributes là hai cơ chế khác nhau.

---

## 11. Browser load SPA nhưng JavaScript vẫn có thể crash

Có thể có:

```text
Network ✅
Nginx ✅
index.html ✅
JS bundle ✅
Application runtime ❌
```

Khi đó cần chuyển sang DevTools Console/Network thay vì tiếp tục sửa Nginx ngẫu nhiên.

---

# Session 5 — Bash, Smoke Test & Week 1 Checkpoint

## 1. Exit code

```bash
echo $?
```

Convention:

```text
0       → success
non-zero → failure
```

CI/CD dựa vào exit code để quyết định step PASS hay FAIL.

Chỉ:

```bash
echo "FAIL"
```

không đủ nếu script vẫn kết thúc bằng `0`.

Phải trả failure thật:

```bash
exit 1
```

---

## 2. `curl -fsS`

```text
-f → HTTP 4xx/5xx làm curl fail
-s → silent progress
-S → vẫn show error
```

Ví dụ:

```bash
curl -fsS http://localhost/not-found
echo $?
```

HTTP error khi dùng `-f` có thể trả:

```text
curl exit 22
```

---

## 3. Hai curl exit code quan trọng

### `22`

```text
đã nhận được HTTP response
nhưng response là 4xx/5xx khi dùng -f
```

### `7`

```text
TCP connection tới host:port không thiết lập được
```

Case:

```text
Nginx /api/healthz → 502 / curl 22
localhost:3000     → curl 7
```

Suy luận:

```text
Nginx reachable
   ↓
upstream :3000 không connect được
   ↓
check Node / systemd / listener
```

---

## 4. Timeout

```bash
curl \
  --connect-timeout 2 \
  --max-time 5 \
  http://localhost/api/healthz
```

```text
--connect-timeout
→ giới hạn thời gian thiết lập connection

--max-time
→ giới hạn toàn bộ request
```

Điều này tránh smoke test hoặc pipeline treo quá lâu khi service bị hang.

---

## 5. Bash strict mode

```bash
set -Eeuo pipefail
```

Ghi nhớ:

```text
-e
→ fail sớm khi command lỗi trong context phù hợp

-u
→ fail khi dùng biến undefined

-o pipefail
→ failure trong pipeline không bị command cuối che mất

-E
→ hỗ trợ ERR trap inheritance tốt hơn
```

---

## 6. Quoting

Sai:

```bash
rm $FILE
```

Tốt hơn:

```bash
rm "${FILE}"
```

Default rule:

> Quote biến Bash trừ khi chủ động muốn word splitting hoặc globbing.

---

## 7. Smoke test script

Core idea:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

BASE_URL="${BASE_URL:-http://127.0.0.1}"

curl \
  -fsS \
  --connect-timeout 2 \
  --max-time 5 \
  "${BASE_URL}/api/healthz" \
  > /dev/null

VERSION="$(curl -fsS "${BASE_URL}/api/version")"

if [[ -z "${VERSION}" ]]; then
  exit 1
fi
```

Mục tiêu:

```text
API khỏe
→ script exit 0

API chết / HTTP lỗi / timeout
→ script non-zero
```

---

## 8. Pipeline failure với `tee`

Case nguy hiểm:

```bash
npm test | tee test.log
```

Nếu:

```text
npm test = 1
tee      = 0
```

mà không bật `pipefail`, shell có thể coi pipeline là success.

Fix:

```bash
set -o pipefail
npm test | tee test.log
```

Điểm quan trọng:

> `pipefail` phải được bật trong shell tạo ra pipeline có ký tự `|`.

Ví dụ:

```bash
set -o pipefail
./smoke-test.sh | tee smoke.log
```

Chỉ bật `pipefail` bên trong `smoke-test.sh` không tự động thay đổi shell cha đang thực thi pipeline.

---

# Unified Debug Mental Model

Đây là phần quan trọng nhất sau 5 buổi.

Khi có incident, không bắt đầu bằng restart.

Bắt đầu bằng:

```text
Symptom
   ↓
Evidence
   ↓
Layer localization
   ↓
Hypothesis
   ↓
Verification
   ↓
Root cause
   ↓
Fix
   ↓
Verify toàn flow
```

---

## Case 1 — DNS lỗi

Symptom:

```text
Could not resolve host
```

Check:

```bash
getent hosts <hostname>
dig <hostname>
```

Kết luận:

```text
DNS fail
TCP/TLS/HTTP chưa bắt đầu
```

---

## Case 2 — Connection refused

Check:

```bash
ss -ltnp | grep :<port>
systemctl status <service>
```

Thường nghi:

```text
wrong port
no listener
process chết
active reject
```

---

## Case 3 — Timeout

Thường nghi:

```text
route
firewall
security rule
packet filtering
network path
```

---

## Case 4 — TLS error

Nếu TCP đã connect rồi mới có certificate error:

```text
DNS/TCP/port = OK
TLS = FAIL
HTTP = chưa tới
```

---

## Case 5 — HTTP 404

Nếu nhận HTTP `404`:

```text
TCP/HTTP server đã hoạt động
```

Ưu tiên:

```text
route/path
proxy_pass URI rewrite
application endpoint
```

---

## Case 6 — HTTP 500

```text
request đã tới application
```

Ưu tiên:

```text
application logs
config
database/cache/external dependency
```

---

## Case 7 — HTTP 502

```text
client → Nginx OK
Nginx → upstream có vấn đề
```

Check:

```bash
curl http://127.0.0.1:3000/healthz
ss -ltnp | grep :3000
sudo systemctl status myapp
sudo journalctl -u myapp -n 50 --no-pager
sudo tail -n 50 /var/log/nginx/error.log
```

Nếu Node trực tiếp OK nhưng qua Nginx vẫn `502`:

```bash
sudo nginx -t
sudo nginx -T
sudo nginx -T | grep -n "proxy_pass"
```

Ưu tiên kiểm tra:

```text
upstream host
port
protocol
proxy_pass
loaded config
```

---

# Command Cheat Sheet — Week 1

## Identity / filesystem

```bash
whoami
id
pwd
ls -la
```

Permission:

```bash
sudo chown root:appuser /opt/myapp/config.json
sudo chmod 640 /opt/myapp/config.json
sudo -u appuser cat /opt/myapp/config.json
```

---

## Process

```bash
ps aux | head
pgrep -af node
```

---

## Port / listener

```bash
ss -ltnp
ss -ltnp | grep :3000
ss -ltnp | grep :80
ss -ltnp | grep :443
```

---

## systemd

```bash
sudo systemctl status myapp --no-pager
sudo systemctl start myapp
sudo systemctl stop myapp
sudo systemctl restart myapp
sudo systemctl enable myapp
sudo systemctl daemon-reload
```

---

## Logs

```bash
sudo journalctl -u myapp -n 50 --no-pager
sudo journalctl -u myapp -f
sudo journalctl -u myapp --since "10 minutes ago"
```

---

## Network

```bash
ip addr
ip route
ip route get <destination-ip>
```

DNS:

```bash
getent hosts <hostname>
dig <hostname>
dig +short <hostname>
```

---

## HTTP / TCP / TLS

```bash
curl -v http://host:port/path
curl -v https://host/path
curl -fsS http://host/path
curl --connect-timeout 2 --max-time 5 http://host/path
```

Bypass DNS:

```bash
curl -vk --resolve hostname:443:IP https://hostname/path
```

---

## Nginx

```bash
sudo systemctl status nginx --no-pager
sudo nginx -t
sudo nginx -T
sudo nginx -T | grep -n "proxy_pass"
sudo systemctl reload nginx
sudo tail -n 50 /var/log/nginx/error.log
```

---

## Bash / CI

```bash
echo $?
chmod +x smoke-test.sh
set -Eeuo pipefail
```

Pipeline safety:

```bash
set -o pipefail
npm test | tee test.log
```

---

# Week 1 Checkpoint

Sau 5 buổi cần tự giải thích được toàn bộ flow:

```text
Browser
   ↓
DNS
   ↓
IP / Route
   ↓
TCP
   ↓
Port / Listener
   ↓
TLS
   ↓
HTTP
   ↓
Nginx
   ↓
Node
   ↓
Dependency
```

Và tự khoanh vùng được ít nhất các incident sau:

```text
DNS sai
port sai
process chết
permission sai
service restart loop
connection refused
timeout
TLS certificate error
HTTP 404
HTTP 500
HTTP 502
Nginx proxy_pass sai
SPA deep route 404
browser JS runtime crash
CI test fail nhưng bị tee che exit code
```

Debug flow cần trở thành phản xạ:

```text
1. Quan sát symptom.
2. Xác định layer gần nhất đã có evidence thành công.
3. Kiểm tra layer kế tiếp bằng command phù hợp.
4. Đặt hypothesis.
5. Verify hypothesis bằng evidence.
6. Chỉ sửa sau khi có evidence đủ mạnh.
7. Verify trực tiếp component bị lỗi.
8. Verify lại qua public/user path.
9. Chạy smoke test.
10. Xác nhận exit code đúng cho automation.
```

Không debug kiểu:

```text
error
 ↓
restart everything
 ↓
chmod 777
 ↓
change firewall
 ↓
try again
```

Mà phải debug kiểu:

```text
error
 ↓
collect evidence
 ↓
localize layer
 ↓
identify root cause
 ↓
minimal fix
 ↓
verify
```

---

# Chuẩn bị sang Week 2

Sau tuần 1, nền tảng đã đủ để chuyển sang Docker mà không coi container là một “hộp đen”.

Các khái niệm tuần 1 sẽ được map trực tiếp sang Docker:

```text
Linux process
→ process trong container

Linux user / least privilege
→ Docker USER

port / listener
→ container port

0.0.0.0 vs 127.0.0.1
→ container networking

artifact
→ image

reproducible npm install
→ reproducible image build

Nginx + Node
→ multi-container architecture

system health / smoke test
→ container health / release verification
```

Session tiếp theo:

```text
Session 6 — Docker image có thể dựng lại
```

Trọng tâm:

```text
Dockerfile multi-stage
lockfile
build context
.dockerignore
base image version
reproducible build
image gắn commit SHA
```
