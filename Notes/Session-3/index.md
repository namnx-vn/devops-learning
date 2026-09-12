# Session 3 — Networking for Debugging

## Mục tiêu buổi học

- Hiểu request đi qua các tầng: DNS → route/network path → TCP → port → TLS → HTTP → application.
- Dùng evidence để xác định lỗi nằm ở layer nào thay vì sửa ngẫu nhiên.
- Phân biệt rõ `connection refused`, `timeout`, TLS error và HTTP 5xx.
- Hiểu IP, subnet, route và default gateway ở mức đủ để debug application.
- Dùng `ip`, `ss`, `dig`, `getent`, `curl`, `systemctl`, `journalctl` để khoanh vùng sự cố.
- Hoàn thành checkpoint với 3 incident: DNS sai, port sai, process chết.

Mental model xuyên suốt:

```text
Client
  ↓
DNS
  ↓
Destination IP
  ↓
Route / network path
  ↓
TCP handshake
  ↓
Destination port / listener
  ↓
TLS handshake
  ↓
HTTP
  ↓
Reverse proxy / application / dependency
```

Nguyên tắc chính:

> Layer nào đã có evidence xác nhận thành công thì không quay lại sửa layer đó một cách ngẫu nhiên.

---

## 1. IP, Subnet và Route

Ví dụ máy có:

```text
IP: 10.0.1.20/24
```

`/24` tương đương subnet mask:

```text
255.255.255.0
```

Network:

```text
10.0.1.0/24
```

Nếu destination là:

```text
10.0.1.50
```

thì source `10.0.1.20/24` và destination `10.0.1.50` cùng subnet.

Ví dụ route:

```bash
ip route
```

```text
default via 10.0.1.1 dev eth0
10.0.1.0/24 dev eth0
```

Với destination `10.0.1.50`, Linux dùng route cụ thể:

```text
10.0.1.0/24 dev eth0
```

nên packet đi trực tiếp qua `eth0`, không đi qua default gateway.

Nếu destination là:

```text
8.8.8.8
```

thì không thuộc `10.0.1.0/24`, nên Linux dùng:

```text
default via 10.0.1.1
```

### Command quan trọng

```bash
ip addr
ip route
ip route get 10.0.1.50
```

Mental model:

```text
same subnet
→ direct route

outside subnet
→ default gateway hoặc route cụ thể khác
```

---

## 2. TCP Handshake

TCP connection bình thường:

```text
Client              Server

SYN  ──────────────►
     ◄──────── SYN-ACK
ACK  ──────────────►

TCP established ✅
```

Chỉ sau khi TCP established thì TLS mới có thể bắt đầu với HTTPS.

### Connection refused

Case điển hình:

```text
Client              Server

SYN  ──────────────►
     ◄──────── RST

TCP established ❌
```

Biểu hiện:

```text
Connection refused
```

Điều này không có nghĩa TCP đã connect thành công.

Nó thường cho thấy:

```text
network path tới host có phản hồi
        ↓
TCP connection bị từ chối
        ↓
không có listener phù hợp hoặc bị active reject
```

Ưu tiên kiểm tra:

```bash
ss -ltnp
ss -ltnp | grep :443
systemctl status <service>
journalctl -u <service>
```

### Timeout

Case điển hình:

```text
SYN ──────────────►
...
không nhận được phản hồi phù hợp
```

Biểu hiện:

```text
Connection timed out
```

Ưu tiên nghi:

```text
routing
firewall
security group / ACL
network path
packet filtering
host unreachable
```

Với timeout, chưa thể khẳng định route/network path OK.

---

## 3. Port và Listener

Ví dụ:

```bash
ss -ltnp | grep :3000
```

Có output:

```text
LISTEN ... 0.0.0.0:3000 ... node
```

thì có evidence rằng process đang giữ listener TCP trên port 3000.

Điểm cần nhớ:

```text
Trying 10.0.1.50:443...
```

không có nghĩa port 443 OK.

Chỉ khi thấy kiểu:

```text
Connected to 10.0.1.50 port 443
```

thì mới có evidence TCP connect tới port đó thành công.

Nếu:

```bash
ss -ltnp | grep :443
```

không có output và client nhận `Connection refused`, lỗi gần nhất nằm ở TCP/listener của port 443.

---

## 4. DNS Debugging

Nếu:

```bash
curl https://api.example.com
```

trả:

```text
Could not resolve host: api.example.com
```

thì:

```text
DNS   = ERROR
Route = chưa tới
TCP   = chưa tới
TLS   = chưa tới
HTTP  = chưa tới
```

Kiểm tra:

```bash
dig api.example.com
dig +short api.example.com
```

Nếu muốn xem system resolver thực tế, đặc biệt khi dùng `/etc/hosts`:

```bash
getent hosts api.example.com
```

### DNS resolve được chưa chắc DNS đúng

Ví dụ:

```text
api.example.com → 10.0.1.99
```

nhưng API thật nằm tại:

```text
10.0.1.50
```

thì DNS vẫn là root cause dù symptom có thể là TCP timeout.

Mental model:

```text
DNS trả sai IP
      ↓
client đi nhầm destination
      ↓
TCP có thể timeout
```

Có thể verify bằng cách bypass DNS:

```bash
curl -vk --resolve api.example.com:443:10.0.1.50 \
  https://api.example.com/healthz
```

Nếu bypass DNS hoạt động thì evidence rất mạnh rằng DNS record hiện tại sai.

### `dig` và `/etc/hosts`

Trong lab nếu cố tình sửa `/etc/hosts`, nên dùng:

```bash
getent hosts api.lab.local
```

vì `dig` query DNS server trực tiếp và thường không đọc `/etc/hosts`.

---

## 5. TLS Debugging

Nếu output có:

```text
Connected to api.example.com (10.0.1.20) port 443
```

rồi:

```text
SSL certificate problem: certificate has expired
```

thì:

```text
DNS   = OK
Route = OK
TCP   = OK
443   = OK
TLS   = ERROR
HTTP  = chưa tới
```

Lý do: TCP đã established nhưng certificate validation fail trong TLS.

### Hostname mismatch

Ví dụ client truy cập:

```text
api.example.com
```

nhưng certificate chỉ hợp lệ cho:

```text
www.example.com
```

thì lỗi là:

```text
TLS hostname mismatch
```

Không phải TCP error.

Mental model:

```text
TCP established ✅
        ↓
TLS certificate validation ❌
        ↓
HTTP chưa được gửi
```

---

## 6. HTTP và Application Error

Nếu:

```text
HTTP/1.1 500 Internal Server Error
{"error":"database unavailable"}
```

thì:

```text
DNS         = OK
Route       = OK
TCP         = OK
443         = OK
TLS         = OK
HTTP        = OK
Application = ERROR
```

Không ưu tiên kiểm tra firewall giữa client và API vì client đã nhận được HTTP response.

Nhưng firewall vẫn có thể liên quan ở dependency phía sau:

```text
Browser → API ✅
API → Database ❌
```

Debug tiếp:

```text
HTTP 500
 ↓
application log
 ↓
DB config / credential
 ↓
app → DB network
 ↓
DB process / health
```

---

## 7. HTTP 502 — Reverse Proxy / Upstream

Nếu:

```text
DNS              ✅
TCP :443         ✅
TLS              ✅
HTTP request     ✅
HTTP/1.1 502 Bad Gateway
```

thì lỗi gần nhất nằm ở:

```text
reverse proxy / upstream application
```

Ví dụ:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:3000;
}
```

Debug upstream:

```bash
curl -v http://127.0.0.1:3000/healthz
ss -ltnp | grep :3000
systemctl status myapp
journalctl -u myapp -n 50 --no-pager
```

Root cause chain có thể là:

```text
Browser thấy 502
      ↓
Nginx không connect được upstream
      ↓
Node :3000 không có listener
      ↓
myapp chết
      ↓
journalctl
      ↓
root cause thật
```

---

## 8. Cách đọc lỗi theo Layer

### Case: Connection Refused

```text
DNS        có thể OK
Route      thường có phản hồi từ destination/path
TCP        ERROR
Port       không accept connection
TLS        chưa tới
HTTP       chưa tới
```

Ưu tiên:

```text
listener
process
service
```

### Case: Timeout

```text
TCP handshake không hoàn tất
```

Ưu tiên:

```text
route
firewall
security rules
network path
```

### Case: TLS Error

```text
DNS        OK
Route      OK
TCP        OK
Port       OK
TLS        ERROR
HTTP       chưa tới
```

### Case: HTTP 500

```text
network/TLS/HTTP path đã hoạt động
application hoặc dependency lỗi
```

### Case: HTTP 502

```text
client → reverse proxy OK
reverse proxy → upstream có vấn đề
```

---

# 9. Incident Lab 1 — DNS Sai

## Mục tiêu

Tạo symptom do hostname resolve sai IP, sau đó dùng evidence chứng minh root cause nằm ở name resolution.

## Cách tái hiện

Đảm bảo API thật đang chạy:

```bash
sudo systemctl start myapp
curl http://127.0.0.1:3000/healthz
```

Backup hosts file:

```bash
sudo cp /etc/hosts /etc/hosts.backup
```

Cố tình map hostname sai:

```bash
echo "203.0.113.123 api.lab.local" | sudo tee -a /etc/hosts
```

Kiểm tra:

```bash
getent hosts api.lab.local
```

Expected:

```text
203.0.113.123 api.lab.local
```

Gọi API:

```bash
curl -v --connect-timeout 3 http://api.lab.local:3000/healthz
```

Có thể timeout hoặc unreachable tùy môi trường.

## Debug

```bash
getent hosts api.lab.local
curl -v http://127.0.0.1:3000/healthz
```

Nếu IP trực tiếp trả `200 OK` nhưng hostname đi sai IP:

```text
Application       ✅
Port 3000         ✅
Hostname mapping  ❌
```

## Fix

```bash
sudo sed -i '/api\.lab\.local/d' /etc/hosts
echo "127.0.0.1 api.lab.local" | sudo tee -a /etc/hosts
```

Verify:

```bash
getent hosts api.lab.local
curl -v http://api.lab.local:3000/healthz
```

---

# 10. Incident Lab 2 — Sai Port

## Mục tiêu

Chứng minh `Connection refused` không tự động có nghĩa application chết.

## Cách tái hiện

Đảm bảo app chạy trên port 3000:

```bash
sudo systemctl start myapp
curl http://127.0.0.1:3000/healthz
```

Cố tình gọi sai port:

```bash
curl -v http://127.0.0.1:3001/healthz
```

Expected:

```text
Connection refused
```

## Debug

```bash
ss -ltnp | grep :300
pgrep -af node
systemctl status myapp --no-pager
```

Evidence:

```text
Node process       ✅
Listener :3000     ✅
Listener :3001     ❌
```

Root cause:

```text
client gọi sai port
```

## Fix

```bash
curl -v http://127.0.0.1:3000/healthz
```

---

# 11. Incident Lab 3 — Process Chết

## Mục tiêu

Tạo symptom giống sai port nhưng root cause thực tế là process/service không chạy.

## Cách tái hiện đơn giản

Ban đầu:

```bash
sudo systemctl start myapp
curl http://127.0.0.1:3000/healthz
ss -ltnp | grep :3000
```

Cố tình stop service:

```bash
sudo systemctl stop myapp
```

Test lại:

```bash
curl -v http://127.0.0.1:3000/healthz
```

Expected:

```text
Connection refused
```

## Debug

```bash
ss -ltnp | grep :3000
pgrep -af node
systemctl status myapp --no-pager
```

Evidence:

```text
client gọi đúng :3000
listener :3000 không còn
Node process không còn
service inactive/failed
```

## Khôi phục

```bash
sudo systemctl start myapp
ss -ltnp | grep :3000
curl -v http://127.0.0.1:3000/healthz
```

---

# 12. Incident Lab 3 Nâng Cao — Process Crash Vì Permission

Case này sát incident thực tế hơn `systemctl stop`.

Giả sử application đọc:

```text
/opt/myapp/config.json
```

Cố tình phá permission:

```bash
sudo chown root:root /opt/myapp/config.json
sudo chmod 600 /opt/myapp/config.json
```

Service chạy bằng:

```ini
User=appuser
```

Restart:

```bash
sudo systemctl restart myapp
```

Symptom:

```bash
curl -v http://127.0.0.1:3000/healthz
```

có thể trả:

```text
Connection refused
```

Debug:

```bash
ss -ltnp | grep :3000
systemctl status myapp --no-pager
sudo journalctl -u myapp -n 30 --no-pager
```

Root cause chain:

```text
Connection refused
      ↓
no listener :3000
      ↓
Node process chết
      ↓
startup failed
      ↓
EACCES config.json
      ↓
wrong permission / ownership
```

Fix theo least privilege:

```bash
sudo chown root:appuser /opt/myapp/config.json
sudo chmod 640 /opt/myapp/config.json
sudo systemctl restart myapp
```

Verify:

```bash
sudo -u appuser cat /opt/myapp/config.json
ss -ltnp | grep :3000
curl http://127.0.0.1:3000/healthz
```

---

## 13. Ba Incident Cần Nhớ

```text
1. DNS sai
hostname → sai IP
getent/dig
        ↓
fix name resolution

2. Port sai
process sống
:3000 LISTEN
client gọi :3001
        ↓
fix client/config port

3. Process chết
client gọi đúng :3000
nhưng không còn listener
        ↓
systemctl
        ↓
journalctl
        ↓
root cause
```

Điểm quan trọng:

```text
              Connection refused
                     │
          ┌──────────┴──────────┐
          │                     │
       sai port             process chết
          │                     │
:3000 vẫn LISTEN       :3000 không LISTEN
          │                     │
      ss chứng minh         ss chứng minh
```

Cùng một symptom có thể có root cause khác nhau.

---

## 14. Debug Flow Chuẩn

```text
Observation
   ↓
Xác định layer gần nhất
   ↓
Hypothesis
   ↓
Command lấy evidence
   ↓
Verify / reject hypothesis
   ↓
Root cause
   ↓
Fix
   ↓
Verify lại toàn flow
```

Không làm:

```text
error
 ↓
restart ngẫu nhiên
 ↓
sửa firewall
 ↓
chmod 777
 ↓
thử lại
```

Nên làm:

```text
DNS?
 ↓
Route?
 ↓
TCP?
 ↓
Port/listener?
 ↓
TLS?
 ↓
HTTP?
 ↓
Reverse proxy?
 ↓
Application/dependency?
```

---

## 15. Command Cheat Sheet

```bash
# IP / interface
ip addr

# Route
ip route
ip route get <destination-ip>

# DNS
getent hosts <hostname>
dig <hostname>
dig +short <hostname>

# Listener / port
ss -ltnp
ss -ltnp | grep :3000
ss -ltnp | grep :443

# Process
pgrep -af node

# HTTP/TCP/TLS evidence
curl -v http://host:port/path
curl -v https://host/path
curl --connect-timeout 3 http://host:port/path

# Bypass DNS while preserving hostname/TLS SNI
curl -vk --resolve hostname:443:IP https://hostname/path

# Service
systemctl status myapp --no-pager

# Logs
journalctl -u myapp -n 50 --no-pager
```

---

## 16. Checkpoint Buổi 3

Đạt checkpoint khi có thể tự giải thích và debug bằng evidence:

```text
DNS sai
port sai
process chết
```

Đồng thời phân biệt được:

```text
Connection refused
→ TCP connection bị từ chối; ưu tiên listener/process.

Timeout
→ TCP handshake không hoàn tất; ưu tiên route/firewall/network path.

TLS error
→ TCP/port đã OK, lỗi trong TLS/certificate validation.

HTTP 500
→ request đã tới application; app/dependency lỗi.

HTTP 502
→ request đã tới reverse proxy; upstream có vấn đề.
```

Mental model cuối buổi:

```text
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
Reverse Proxy
 ↓
Application
 ↓
Dependencies
```

> Debug tốt không phải là biết thật nhiều command. Quan trọng là biết mỗi command đang kiểm chứng hypothesis nào và evidence đó loại bỏ được layer nào.
