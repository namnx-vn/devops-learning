# Session 2 — Linux for Web Applications, systemd & Debugging

## Mục tiêu buổi học

- Hiểu mối quan hệ giữa Linux user, file permission, process và port.
- Chạy một Node.js API bằng user thường thay vì `root`.
- Hiểu nguyên tắc **least privilege** trong vận hành application.
- Biến Node.js application thành Linux service bằng `systemd`.
- Phân biệt `start`, `stop`, `restart`, `enable` và `daemon-reload`.
- Đọc log service bằng `journalctl`.
- Debug lỗi application theo evidence thay vì sửa ngẫu nhiên.
- Phân biệt process crash, application unhealthy và network/connectivity issue.

Mental model xuyên suốt buổi học:

```text
Node source
   ↓
Linux user
   ↓
file permission
   ↓
process
   ↓
port
   ↓
systemd
   ↓
journal/log
   ↓
debug incident
```

---

## 1. Linux User và Process Identity

Khi chạy:

```bash
node server.js
```

Node process chạy với identity của Linux user đã thực thi command.

Ví dụ:

```bash
whoami
```

trả về:

```text
ubuntu
```

thì process Node được chạy dưới user `ubuntu`.

Trong production không nên chạy application bằng `root` nếu application không thật sự cần quyền root.

### Least Privilege

Nguyên tắc:

> Một process chỉ nên được cấp đúng những quyền tối thiểu cần thiết để thực hiện nhiệm vụ của nó.

Ví dụ:

```text
root
 └─ quyền rất lớn trên hệ thống

appuser
 └─ chỉ có quyền cần thiết để chạy application
```

Nếu application bị khai thác RCE:

```text
App chạy bằng root
        ↓
attacker có thể nhận quyền rất lớn

App chạy bằng appuser
        ↓
impact bị giới hạn bởi permission của appuser
```

Concept này sẽ gặp lại ở nhiều nơi:

```text
Linux user
AWS IAM
Docker USER
Kubernetes ServiceAccount
Jenkins credentials
```

---

## 2. Linux File Permission

Ví dụ:

```text
-rw------- 1 root root config.json
```

Permission được chia thành:

```text
- rw- --- ---
  │   │   │
  │   │   └── other
  │   └────── group
  └────────── owner
```

Các quyền:

```text
r = read
w = write
x = execute
```

Giá trị số:

```text
r = 4
w = 2
x = 1
```

Ví dụ:

```text
7 = rwx
6 = rw-
5 = r-x
4 = r--
```

### File permission và ownership phải đọc cùng nhau

Ví dụ:

```text
-rw------- root root config.json
```

có nghĩa:

```text
owner root → read + write
group root → no permission
other      → no permission
```

Nếu Node chạy bằng `appuser`, application không thể đọc file này.

Một điểm đã sửa trong quá trình học:

```text
"file có rw nên Node đọc được" ❌
```

Phải xác định `rw` thuộc owner/group/other nào và process đang chạy bằng user nào.

---

## 3. Node API dùng trong Lab

Application tối giản có các endpoint:

```text
/healthz
/version
/error
```

Mục đích:

```text
/healthz → kiểm tra application health
/version → xác định version đang chạy
/error   → tạo HTTP 500 có chủ đích để debug
```

Node listen trên:

```text
0.0.0.0:3000
```

Mental model:

```text
server.js
   ↓
Node runtime
   ↓
Linux process
   ↓
PID
   ↓
TCP socket
   ↓
:3000
```

Source code tự nó không giữ port. Process Node được tạo ra từ source code mới là thứ giữ socket/listener.

---

## 4. Process và Port

Một số command dùng trong lab:

```bash
pgrep -af node
```

hoặc:

```bash
ps aux | grep node
```

để tìm process.

Kiểm tra listener:

```bash
ss -ltnp
```

hoặc:

```bash
ss -ltnp | grep 3000
```

Ví dụ:

```text
LISTEN ... 0.0.0.0:3000 ... node
```

### `127.0.0.1` vs `0.0.0.0`

```text
127.0.0.1:3000
```

chỉ nhận connection qua loopback của chính server.

```text
0.0.0.0:3000
```

listen trên mọi IPv4 interface của server.

Nếu:

```text
curl localhost:3000 → success
curl VM-IP:3000     → fail
```

không nên kết luận Node chết ngay.

Có thể kiểm tra:

```text
bind address
firewall
routing
security rules
network
```

---

## 5. Connection Refused vs Timeout

Khi:

```bash
curl localhost:3000
```

trả:

```text
Connection refused
```

ưu tiên kiểm tra process/listener trước:

```bash
systemctl status myapp
pgrep -af node
ss -ltnp | grep 3000
```

Mental model thường dùng:

```text
Connection refused
       ↓
host reachable
       ↓
không có listener phù hợp tại port
```

Có thể do:

```text
process chết
service chưa start
app listen port khác
bind address không phù hợp
```

Trong khi timeout thường khiến ta nghi nhiều hơn tới:

```text
firewall
routing
security group
network path
```

Đây là heuristic debug, không phải quy luật tuyệt đối cho mọi trường hợp.

---

## 6. Dedicated Application User

Tạo application user riêng:

```bash
sudo useradd \
  --system \
  --create-home \
  --shell /usr/sbin/nologin \
  appuser
```

Kiểm tra:

```bash
id appuser
```

Application được đặt tại:

```text
/opt/myapp
```

Mục tiêu:

```text
systemd
   ↓
User=appuser
   ↓
Node process
```

Không sử dụng `root` nếu application không cần quyền root.

---

## 7. systemd Service

Unit file:

```text
/etc/systemd/system/myapp.service
```

Ví dụ cấu hình:

```ini
[Unit]
Description=DevOps Learning Node API
After=network.target

[Service]
Type=simple
User=appuser
Group=appuser
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/node /opt/myapp/server.js
Environment=PORT=3000
Environment=APP_VERSION=v1.0.0
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
```

### Các command quan trọng

Start service ngay:

```bash
sudo systemctl start myapp
```

Stop service:

```bash
sudo systemctl stop myapp
```

Restart service:

```bash
sudo systemctl restart myapp
```

Cho phép tự start khi machine boot:

```bash
sudo systemctl enable myapp
```

Enable và start ngay:

```bash
sudo systemctl enable --now myapp
```

### `start` khác `enable`

```text
systemctl start
    ↓
start service ngay lúc này

systemctl enable
    ↓
configure service để tự start khi boot
```

`enabled` không có nghĩa service đang healthy hoặc thậm chí đang running.

Ví dụ có thể có:

```text
enabled ✅
running ❌
healthy ❌
```

---

## 8. `daemon-reload`

Khi sửa:

```text
/etc/systemd/system/myapp.service
```

cần chạy:

```bash
sudo systemctl daemon-reload
```

để systemd đọc lại unit definition.

Sau đó nếu muốn process hiện tại áp dụng configuration mới:

```bash
sudo systemctl restart myapp
```

Mental model:

```text
edit unit file
      ↓
daemon-reload
      ↓
systemd reload unit definition
      ↓
restart service
      ↓
new process sử dụng config mới
```

Điểm cần nhớ:

```text
daemon-reload ≠ restart application
```

`daemon-reload` chỉ reload definition của systemd.

---

## 9. `systemctl status` và `journalctl`

Kiểm tra trạng thái service:

```bash
systemctl status myapp
```

Các phần quan trọng:

```text
Loaded
Active
Main PID
Result / exit code
```

Ví dụ lab thực tế:

```text
Active: activating (auto-restart) (Result: exit-code)
Main PID: ... (code=exited, status=1/FAILURE)
```

Điều này cho thấy:

```text
systemd start Node
        ↓
Node exit code 1
        ↓
Restart=on-failure
        ↓
auto restart
```

### journalctl

Đọc log service:

```bash
sudo journalctl -u myapp
```

30 dòng gần nhất:

```bash
sudo journalctl -u myapp -n 30 --no-pager
```

Follow realtime:

```bash
sudo journalctl -u myapp -f
```

Log từ boot hiện tại:

```bash
sudo journalctl -u myapp -b
```

Mental model:

```text
Node stdout / stderr
        ↓
systemd journal
        ↓
journalctl
```

---

## 10. Lab thực tế — EACCES Permission Denied

Service gặp lỗi:

```text
Error: EACCES: permission denied, open '/opt/myapp/config.json'
```

File permission:

```text
-rw------- 1 root root /opt/myapp/config.json
```

Service chạy bằng:

```ini
User=appuser
```

### Chuỗi nguyên nhân

```text
systemd start myapp
        ↓
Node chạy bằng appuser
        ↓
server.js đọc config.json
        ↓
config.json thuộc root:root mode 600
        ↓
appuser không có read permission
        ↓
EACCES
        ↓
Node exit 1
        ↓
systemd restart
        ↓
Node lại fail
        ↓
restart loop
```

Trong lab restart counter đã tăng tới hơn 80 lần, minh họa rõ crash/restart loop.

### Verify hypothesis bằng identity thật của service

Command:

```bash
sudo -u appuser cat /opt/myapp/config.json
```

kết quả:

```text
Permission denied
```

Đây là evidence trực tiếp rằng `appuser` không thể đọc file.

Flow debug đúng:

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

Không nên làm:

```text
error
 ↓
chmod 777
 ↓
restart ngẫu nhiên
```

---

## 11. Fix Permission theo Least Privilege

Một fix có thể chạy được:

```bash
sudo chown appuser:appuser /opt/myapp/config.json
sudo chmod 600 /opt/myapp/config.json
```

Nhưng nếu application chỉ cần **read** config, permission tốt hơn là:

```bash
sudo chown root:appuser /opt/myapp/config.json
sudo chmod 640 /opt/myapp/config.json
```

Kết quả:

```text
-rw-r----- root appuser config.json
```

Quyền:

```text
root    → read + write
appuser → read only
others  → no permission
```

Đây là least privilege chính xác hơn vì application không được cấp quyền write nếu không cần.

### Không dùng `chmod 777` chỉ để app chạy

```text
777
owner → rwx
group → rwx
other → rwx
```

Điều này mở quyền không cần thiết và tăng attack surface.

Câu hỏi đúng không chỉ là:

> Làm sao để app chạy?

mà là:

> Quyền tối thiểu nào giúp app chạy đúng?

---

## 12. Root Cause vs Symptom

Trong lab:

```text
curl localhost:3000
→ Connection refused
```

Đây chỉ là symptom.

Root cause chain:

```text
Connection refused
        ↓
không có listener :3000
        ↓
Node process chết
        ↓
Node startup failed
        ↓
readFileSync(config.json) failed
        ↓
EACCES permission denied
        ↓
wrong file ownership / permission
```

Điểm quan trọng:

> Symptom có thể xuất hiện ở một layer nhưng root cause nằm ở layer khác.

Không nên thấy port lỗi rồi lập tức sửa firewall hoặc restart ngẫu nhiên.

---

## 13. Process Alive không đồng nghĩa Application Healthy

Có thể xảy ra:

```text
systemctl status → active (running)
process          → exists
port :3000       → LISTEN
/healthz         → HTTP 500
```

Khi đó:

```text
process health ✅
application health ❌
```

`systemd` mặc định chủ yếu biết process còn sống hay đã exit; nó không tự hiểu rằng HTTP `/healthz` trả `500` nghĩa application unhealthy.

Application có thể unhealthy vì:

```text
bad configuration
database unavailable
Redis unavailable
external dependency failure
internal application error
```

Concept này sẽ nối trực tiếp tới Kubernetes:

```text
startupProbe
readinessProbe
livenessProbe
```

---

## 14. Restart Policy

Service dùng:

```ini
Restart=on-failure
RestartSec=3
```

Nếu process bị fail theo điều kiện systemd xem là failure, systemd sẽ schedule restart.

Ví dụ lab với process bị `SIGKILL`:

```text
Node PID A
   ↓
SIGKILL
   ↓
process failure
   ↓
systemd detects failure
   ↓
Restart=on-failure
   ↓
Node PID B
```

`systemctl stop myapp` là stop có chủ đích, nên service không được xem như một crash cần auto-restart.

Lưu ý: behavior chính xác của `Restart=on-failure` còn phụ thuộc loại exit status/signal. Trong lab dùng `kill -9` (`SIGKILL`) để tạo failure rõ ràng.

### Restart loop

Nếu app luôn fail ngay khi start:

```text
start
 ↓
crash
 ↓
RestartSec
 ↓
start
 ↓
crash
 ↓
...
```

Restart loop có thể gây:

```text
log spam
CPU churn
alert noise
che mất lỗi ban đầu quan trọng
```

Đây là concept tương tự lỗi `CrashLoopBackOff` sẽ gặp trong Kubernetes.

---

## 15. Debug Flow chuẩn hóa

Khi production báo:

```text
API DOWN
```

không nên sửa ngay.

Flow ưu tiên:

```text
1. Xác định symptom
   ↓
curl / HTTP status / timeout / refused

2. Service status
   ↓
systemctl status

3. Process
   ↓
ps / pgrep

4. Listener
   ↓
ss -ltnp

5. Logs
   ↓
journalctl

6. Đặt hypothesis
   ↓
permission / config / code / dependency / network

7. Verify hypothesis
   ↓
sudo -u appuser ... / curl / ls -l / etc.

8. Fix

9. Verify lại từ dưới lên và từ góc nhìn user
```

Nguyên tắc:

> Evidence trước, thay đổi sau.

---

## 16. Checkpoint Scenarios

### Scenario A

```text
systemctl status myapp → active (running)
ss -ltnp               → :3000 LISTEN
curl /healthz           → HTTP 500
```

Kết luận:

```text
Application-level problem ✅
```

Process và TCP listener đang tồn tại nên chưa có evidence cho thấy systemd hoặc local TCP connectivity là root cause.

Tiếp tục điều tra:

```text
application logs
configuration
database/cache/external dependencies
recent deployment
```

### Scenario B

```text
systemctl status → failed
journalctl       → EACCES permission denied
```

Kết luận:

```text
Linux filesystem permission / ownership problem ✅
```

Cụ thể `appuser` thiếu permission cần thiết để đọc resource mà application cần.

### Scenario C

```text
systemctl status            → active
curl localhost:3000/healthz → success
browser → VM-IP:3000        → timeout
```

Không nên restart Node.

Ta đã có evidence:

```text
application process ✅
local listener/application ✅
```

Nên chuyển sang kiểm tra:

```text
bind address
VM firewall
routing
host networking
cloud security rules nếu có
```

Nếu listener chỉ là:

```text
127.0.0.1:3000
```

thì remote client không thể truy cập trực tiếp qua `VM-IP:3000`.

### Scenario D

Node process bị kill ngoài ý muốn và failure đó thỏa điều kiện `Restart=on-failure`.

Kết quả:

```text
systemd schedules restart ✅
```

Trong lab, dùng `kill -9` tạo `SIGKILL`, nên systemd coi đây là abnormal failure và restart process theo policy. Theo tài liệu systemd, `Restart=on-failure` được thiết kế để restart service khi process kết thúc do failure; không nên hiểu đơn giản rằng mọi signal/exit đều luôn restart giống nhau.

---

## 17. Self Assessment — Session 2

### Tự làm / đã hiểu

- [x] Hiểu process chạy dưới Linux user nào thì bị giới hạn bởi permission của user đó.
- [x] Hiểu nguyên tắc least privilege.
- [x] Hiểu tại sao application không nên chạy bằng root nếu không cần.
- [x] Đọc được owner/group/other trong Linux file permission.
- [x] Hiểu `600`, `640`, `644`, `777` ở mức cần thiết cho lab.
- [x] Hiểu process Node mới là thứ giữ port, không phải source file.
- [x] Biết dùng `pgrep` / `ps` để tìm process.
- [x] Biết dùng `ss -ltnp` để kiểm tra listener.
- [x] Hiểu `127.0.0.1` khác `0.0.0.0`.
- [x] Hiểu `systemctl start` khác `systemctl enable`.
- [x] Hiểu mục đích của `systemctl daemon-reload`.
- [x] Biết dùng `systemctl status` để xem trạng thái service.
- [x] Biết dùng `journalctl -u` để tìm application/service log.
- [x] Đã debug thực tế lỗi `EACCES` bằng evidence.
- [x] Biết verify permission bằng chính service identity với `sudo -u appuser`.
- [x] Hiểu tại sao không nên dùng `chmod 777` làm default fix.
- [x] Hiểu `active (running)` không đảm bảo application healthy.
- [x] Hiểu `Connection refused` có thể chỉ là symptom của process startup failure.
- [x] Hiểu `Restart=on-failure` và restart loop ở mức application operations.
- [x] Phân loại đúng 4 checkpoint scenarios cuối buổi.

### Điểm đã sửa trong quá trình học

- [x] Không thể nhìn `rw` riêng lẻ; phải kết hợp owner/group/other và process identity.
- [x] Với `localhost:3000 → Connection refused`, nên kiểm tra process/listener trước khi nghi firewall.
- [x] `enabled` không đồng nghĩa `running` hoặc `healthy`.
- [x] `daemon-reload` không restart application.
- [x] Process alive không đồng nghĩa application healthy.
- [x] Không restart application khi local health check thành công nhưng remote connection timeout; cần chuyển sang network/bind/firewall investigation.

### Cần học tiếp

- [ ] IP address và network interface.
- [ ] Subnet và route.
- [ ] DNS resolution.
- [ ] TCP connection establishment.
- [ ] TLS handshake.
- [ ] Phân biệt DNS failure, timeout, connection refused, TLS error và HTTP 5xx bằng lab.
- [ ] Dùng `ip`, `ss`, `dig`, `curl` để debug networking có hệ thống.

---

## 18. Kết luận Session 2

Buổi 2 đã đạt checkpoint của plan:

```text
API chạy bằng user thường
        ✅
Hiểu file permission
        ✅
Hiểu process và port
        ✅
Dùng systemd quản lý app
        ✅
Dùng journalctl đọc lỗi
        ✅
Tự tìm root cause EACCES
        ✅
Fix theo least privilege
        ✅
Phân biệt process health và app health
        ✅
```

Mental model quan trọng nhất sau buổi học:

```text
User reports API failure
          ↓
Observe symptom
          ↓
Service status
          ↓
Process
          ↓
Port / listener
          ↓
Logs
          ↓
Hypothesis
          ↓
Verify
          ↓
Fix
          ↓
Verify again
```

Và chuỗi hệ thống:

```text
Source Code
    ↓
Linux User / Permission
    ↓
Node Process
    ↓
TCP Port
    ↓
systemd
    ↓
journalctl
    ↓
Operational Debugging
```

## Next Session

**Session 3 — Networking để debug**

Theo plan:

```text
IP
→ subnet
→ route
→ DNS
→ TCP
→ TLS
→ HTTP
```

Lab tiếp theo sẽ tập trung phân biệt và tạo evidence cho:

```text
DNS failure
connection refused
timeout
TLS failure
HTTP 5xx
```

Mục tiêu là nhìn một lỗi connection và xác định được nó đang xảy ra ở layer nào thay vì restart application theo cảm tính.
