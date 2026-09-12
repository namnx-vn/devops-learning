# Debug Command Reference — DNS → Route → TCP → Port → TLS → HTTP → Application

File này là command reference dùng xuyên suốt các buổi DevOps.

Mục tiêu không phải nhớ thật nhiều command, mà là biết:

```text
triệu chứng
  ↓
đang lỗi ở layer nào?
  ↓
command nào tạo ra evidence?
  ↓
đọc output thế nào?
  ↓
command đó chứng minh được gì / chưa chứng minh được gì?
```

Mental model chung:

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
Port / listener
  ↓
TLS
  ↓
HTTP
  ↓
Reverse proxy
  ↓
Application
  ↓
Dependency: DB / Redis / external API
```

> Nguyên tắc: layer nào đã có evidence xác nhận thành công thì không quay lại sửa layer đó một cách ngẫu nhiên.

---

# 1. Local machine / interface / IP

## `ip addr`

```bash
ip addr
```

Dạng ngắn:

```bash
ip a
```

Dùng để xem:

- network interface hiện có;
- IPv4/IPv6 của máy;
- subnet prefix như `/24`;
- interface đang `UP` hay `DOWN`.

Ví dụ:

```text
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 10.0.1.20/24 brd 10.0.1.255 scope global eth0
```

Đọc như sau:

```text
eth0          = network interface
10.0.1.20     = IPv4 của máy
/24           = network prefix
UP            = interface được enable
LOWER_UP      = link layer đang hoạt động
```

`10.0.1.20/24` thuộc network:

```text
10.0.1.0/24
```

Command này giúp trả lời:

> Máy hiện tại có IP nào và request có thể đi ra interface nào?

Nó chưa chứng minh destination reachable.

---

## `ip link`

```bash
ip link
```

Dùng để kiểm tra trạng thái interface ở layer thấp hơn IP.

Ví dụ:

```text
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP>
```

Nếu interface `DOWN`, routing/IP phía trên sẽ không hoạt động bình thường.

---

# 2. Route / network path

## `ip route`

```bash
ip route
```

Ví dụ:

```text
default via 10.0.1.1 dev eth0
10.0.1.0/24 dev eth0 proto kernel scope link src 10.0.1.20
```

Ý nghĩa:

```text
10.0.1.0/24 dev eth0
→ destination thuộc subnet này thì gửi trực tiếp qua eth0

default via 10.0.1.1 dev eth0
→ destination không match route cụ thể hơn thì gửi cho gateway 10.0.1.1
```

Các từ thường gặp:

```text
default = route fallback
via     = next-hop / gateway
dev     = interface dùng để gửi packet
src     = source IP Linux ưu tiên sử dụng
```

---

## `ip route get <IP>`

Một trong những command hữu ích nhất khi debug route:

```bash
ip route get 10.0.1.50
```

Ví dụ:

```text
10.0.1.50 dev eth0 src 10.0.1.20
```

Nghĩa là Linux định gửi packet:

```text
destination = 10.0.1.50
interface   = eth0
source IP   = 10.0.1.20
```

Nếu destination ngoài subnet:

```bash
ip route get 8.8.8.8
```

có thể ra:

```text
8.8.8.8 via 10.0.1.1 dev eth0 src 10.0.1.20
```

Command này trả lời chính xác hơn `ip route` cho câu hỏi:

> Với destination cụ thể này, kernel sẽ chọn route nào?

---

## `ping <host>`

```bash
ping 10.0.1.50
```

Giới hạn số packet:

```bash
ping -c 4 10.0.1.50
```

Flags:

```text
-c 4 = gửi 4 ICMP echo request rồi dừng
```

Nếu nhận reply:

```text
64 bytes from 10.0.1.50 ...
```

thì có evidence rằng IP path và ICMP hoạt động.

Nhưng:

> `ping` fail không đồng nghĩa server chết.

ICMP có thể bị firewall chặn trong khi TCP 443 vẫn hoạt động bình thường.

Không dùng `ping` làm bằng chứng duy nhất.

---

## `tracepath` / `traceroute`

```bash
tracepath 8.8.8.8
```

hoặc nếu đã cài traceroute:

```bash
traceroute 8.8.8.8
```

Dùng để quan sát các hop trên network path.

Hữu ích khi:

- timeout;
- nghi route sai;
- packet chết ở một hop trung gian;
- cần so sánh path giữa hai máy.

Lưu ý: firewall có thể chặn response nên dấu `*` không tự động chứng minh hop đó hỏng.

---

# 3. DNS / name resolution

## `dig <domain>`

```bash
dig api.example.com
```

Dùng để query DNS record.

Phần quan trọng thường đọc:

```text
ANSWER SECTION
api.example.com.  300  IN  A  10.0.1.50
```

Ý nghĩa:

```text
A          = IPv4 record
10.0.1.50  = IP DNS trả về
300        = TTL tính bằng giây
```

---

## `dig +short`

```bash
dig +short api.example.com
```

Output gọn:

```text
10.0.1.50
```

Dùng khi chỉ cần kiểm tra nhanh domain đang resolve thành IP nào.

Điểm rất quan trọng:

```text
DNS resolve được ≠ DNS đúng
```

Ví dụ DNS trả:

```text
10.0.1.99
```

nhưng service thật ở:

```text
10.0.1.50
```

thì DNS vẫn là root cause.

---

## `dig @<dns-server> <domain>`

```bash
dig @8.8.8.8 api.example.com
```

Dùng để query trực tiếp một DNS server cụ thể.

Ví dụ:

```text
dig api.example.com
→ fail

dig @8.8.8.8 api.example.com
→ success
```

Có thể nghi DNS resolver mặc định của máy/configuration đang có vấn đề.

---

## `getent hosts <hostname>`

```bash
getent hosts api.lab.local
```

Khác `dig`, command này sử dụng system name resolution theo cấu hình `/etc/nsswitch.conf`.

Do đó nó có thể đọc mapping từ:

```text
/etc/hosts
DNS
các NSS source khác
```

Dùng đặc biệt tốt khi lab sửa `/etc/hosts`.

Ví dụ:

```bash
getent hosts api.lab.local
```

```text
127.0.0.1 api.lab.local
```

---

## `/etc/resolv.conf`

```bash
cat /etc/resolv.conf
```

Dùng để xem resolver configuration cơ bản.

Ví dụ:

```text
nameserver 127.0.0.53
search example.internal
```

`nameserver` là DNS resolver mà hệ thống đang tham chiếu.

Trên systemd-resolved, `127.0.0.53` thường là local stub resolver chứ không phải DNS upstream cuối cùng.

---

## `resolvectl status`

Trên máy dùng `systemd-resolved`:

```bash
resolvectl status
```

Có thể xem:

- DNS server theo từng interface;
- search domain;
- default DNS route;
- resolver thực tế phía sau `127.0.0.53`.

---

## Bypass DNS bằng `curl --resolve`

```bash
curl -v --resolve api.example.com:443:10.0.1.50 \
  https://api.example.com/healthz
```

Cú pháp:

```text
--resolve HOST:PORT:IP
```

Nghĩa là:

> Khi curl truy cập `api.example.com:443`, không dùng DNS record hiện tại; hãy kết nối tới `10.0.1.50`.

Điều này rất hữu ích để kiểm chứng hypothesis:

```text
DNS record sai?
```

Nếu domain bình thường fail nhưng `--resolve` tới IP đúng lại success, DNS là suspect rất mạnh.

---

# 4. TCP / port / listener

## `ss -ltnp`

```bash
ss -ltnp
```

Flags:

```text
-l = chỉ socket đang LISTEN
-t = TCP
-n = hiển thị IP/port dạng số, không resolve tên service
-p = hiển thị process/PID nếu có quyền
```

Ví dụ:

```text
LISTEN 0 511 0.0.0.0:3000 0.0.0.0:* users:(("node",pid=1234,fd=20))
```

Đọc như sau:

```text
LISTEN         = socket đang nhận TCP connection
0.0.0.0:3000  = listen port 3000 trên mọi IPv4 interface
node           = process giữ socket
pid=1234       = process ID
```

Filter một port:

```bash
ss -ltnp | grep :3000
```

Hoặc 443:

```bash
ss -ltnp | grep :443
```

Nếu không có output:

```text
không có TCP listener khớp filter
```

nhưng vẫn cần kiểm tra process/service để tìm nguyên nhân.

---

## `ss -tn`

```bash
ss -tn
```

Dùng để xem TCP connection hiện tại, không chỉ listener.

Các state thường gặp:

```text
ESTAB       = TCP connection established
SYN-SENT    = client đã gửi SYN, đang chờ phản hồi
SYN-RECV    = server đã nhận SYN và gửi SYN-ACK
TIME-WAIT   = connection đã đóng, kernel giữ state tạm thời
CLOSE-WAIT  = peer đã đóng, local process chưa close socket
```

---

## `nc -vz <host> <port>`

Nếu có netcat:

```bash
nc -vz 10.0.1.50 443
```

Flags:

```text
-v = verbose
-z = scan/connect test, không gửi payload application
```

Success có thể giống:

```text
Connection to 10.0.1.50 443 port [tcp/https] succeeded!
```

Dùng để test TCP port nhanh mà không cần HTTP/TLS hoàn chỉnh.

Nó chứng minh TCP connect được, không chứng minh HTTP application healthy.

---

## `curl -v`

```bash
curl -v https://api.example.com/healthz
```

`-v` = verbose.

Đây là command đặc biệt hữu ích vì output đi qua nhiều layer.

Ví dụ:

```text
* Host api.example.com was resolved.
* IPv4: 10.0.1.50
*   Trying 10.0.1.50:443...
* Connected to api.example.com (10.0.1.50) port 443
* SSL connection using TLSv1.3
> GET /healthz HTTP/1.1
< HTTP/1.1 200 OK
```

Có thể đọc theo layer:

```text
resolved           → DNS có kết quả
Trying IP:443      → bắt đầu TCP attempt
Connected          → TCP establish tới port thành công
SSL/TLS output     → đang/đã thực hiện TLS
> GET              → HTTP request đã gửi
< HTTP/1.1 200     → HTTP response đã nhận
```

Điểm cần nhớ:

```text
Trying 10.0.1.50:443
```

KHÔNG có nghĩa port 443 OK.

Phải thấy:

```text
Connected to ... port 443
```

mới có evidence TCP connection thành công.

---

# 5. Connection refused vs timeout

## Connection refused

Ví dụ:

```bash
curl -v http://127.0.0.1:3001
```

```text
connect to 127.0.0.1 port 3001 failed: Connection refused
```

Mental model thường gặp:

```text
SYN
 ↓
RST / active reject
 ↓
TCP connection không establish
```

Ưu tiên kiểm tra:

```bash
ss -ltnp | grep :3001
pgrep -af node
systemctl status myapp
journalctl -u myapp -n 50 --no-pager
```

Common causes:

- process chết;
- service chưa start;
- gọi sai port;
- process listen port khác;
- active reject.

---

## Timeout

Ví dụ:

```text
Connection timed out
```

Mental model:

```text
SYN
 ↓
không nhận được response phù hợp
```

Ưu tiên kiểm tra:

```bash
ip route get <destination-ip>
ping <destination-ip>          # chỉ là signal phụ
tracepath <destination-ip>
```

và firewall/security rules ở hai phía nếu có quyền.

Timeout khiến ta nghi nhiều hơn tới:

```text
route
firewall drop
security group / ACL
network path
host unreachable
```

Không kết luận process chết chỉ từ timeout.

---

# 6. TLS / certificate

## `curl -v https://...`

```bash
curl -v https://api.example.com
```

Dùng để quan sát TLS trong context thực tế của HTTP request.

Các lỗi thường gặp:

```text
certificate has expired
hostname mismatch
unknown CA
self-signed certificate
TLS protocol/cipher mismatch
```

Nếu output đã có:

```text
Connected to ... port 443
```

rồi mới lỗi certificate, thì:

```text
TCP  = OK
443  = OK
TLS  = ERROR
HTTP = chưa tới hoặc chưa hoàn tất
```

---

## `openssl s_client`

```bash
openssl s_client -connect api.example.com:443 -servername api.example.com
```

Hai option quan trọng:

```text
-connect HOST:PORT
→ TCP/TLS endpoint cần kết nối

-servername api.example.com
→ gửi SNI hostname trong TLS ClientHello
```

SNI rất quan trọng khi một IP phục vụ nhiều HTTPS virtual hosts.

Command này giúp xem:

- certificate chain;
- issuer;
- subject;
- TLS protocol;
- verify result;
- server certificate thực tế được trả về.

---

## Xem certificate ngắn gọn

```bash
openssl s_client \
  -connect api.example.com:443 \
  -servername api.example.com \
  </dev/null 2>/dev/null \
| openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```

Ý nghĩa:

```text
-subject        = certificate được cấp cho ai
-issuer         = CA nào ký
-dates          = notBefore / notAfter
-ext subjectAltName = hostname/IP hợp lệ của certificate
```

Dùng để debug:

```text
certificate expired?
hostname mismatch?
certificate nào đang được server trả về?
```

---

## `curl -k`

```bash
curl -vk https://api.example.com
```

`-k` / `--insecure` bỏ qua certificate verification.

Chỉ nên dùng cho lab/debug hypothesis.

Ví dụ:

```text
curl bình thường → certificate error
curl -k          → HTTP 200
```

Evidence:

```text
TCP/HTTP path có thể hoạt động
certificate validation là vấn đề
```

Không coi `-k` là fix production.

---

# 7. HTTP

## Xem status + headers + body

```bash
curl -i https://api.example.com/healthz
```

`-i` = include response headers trong output.

Ví dụ:

```text
HTTP/1.1 500 Internal Server Error
Content-Type: application/json

{"error":"database unavailable"}
```

Nếu đã nhận HTTP response, thì client đã đi qua TCP và nếu dùng HTTPS thì TLS cũng đã đủ thành công để trao đổi HTTP.

---

## Chỉ lấy headers

```bash
curl -I https://example.com
```

`-I` gửi HEAD request.

Hữu ích khi kiểm tra:

- HTTP status;
- redirect;
- cache headers;
- content type;
- server/proxy headers.

Lưu ý: một số app xử lý HEAD khác GET, nên khi nghi application behavior nên test GET thật.

---

## Follow redirect

```bash
curl -L http://example.com
```

`-L` = follow `3xx Location` redirects.

Muốn xem cả redirect chain:

```bash
curl -vL http://example.com
```

---

## Chỉ in status code

```bash
curl -sS -o /dev/null -w '%{http_code}\n' \
  https://api.example.com/healthz
```

Flags:

```text
-s  = silent progress
-S  = vẫn hiện error khi dùng -s
-o /dev/null = bỏ response body
-w  = custom output format
%{http_code} = HTTP status code
```

Rất hữu ích cho smoke test/script.

---

## Đo timing từng phase của curl

```bash
curl -sS -o /dev/null \
  -w 'dns=%{time_namelookup}\nconnect=%{time_connect}\ntls=%{time_appconnect}\nfirst_byte=%{time_starttransfer}\ntotal=%{time_total}\n' \
  https://api.example.com/healthz
```

Các giá trị:

```text
time_namelookup    = thời gian DNS resolve hoàn tất
time_connect       = thời gian TCP connect hoàn tất
time_appconnect    = thời gian TLS handshake hoàn tất
time_starttransfer = thời gian tới byte đầu tiên của response
time_total         = tổng thời gian request
```

Hữu ích khi request không fail nhưng chậm.

---

# 8. Process / service / application

## `pgrep -af <process>`

```bash
pgrep -af node
```

Flags:

```text
-a = hiện full command line
-f = match cả full command line, không chỉ process name
```

Ví dụ:

```text
1234 /usr/bin/node /opt/myapp/server.js
```

Chứng minh process tồn tại.

Nhưng:

```text
process exists ≠ application healthy
```

---

## `ps aux`

```bash
ps aux | grep node
```

Các cột quan trọng:

```text
USER = user chạy process
PID  = process ID
%CPU = CPU usage
%MEM = memory usage
COMMAND = command
```

Dùng khi cần xem process identity/resource usage nhanh.

---

## `systemctl status <service>`

```bash
systemctl status myapp --no-pager
```

Các phần quan trọng:

```text
Loaded   = unit có load được không / enabled state
Active   = running, failed, inactive...
Main PID = process chính
Result   = kết quả cuối
status   = exit code nếu process fail
```

Ví dụ:

```text
Active: activating (auto-restart) (Result: exit-code)
```

có thể nghĩa:

```text
process start
 ↓
exit lỗi
 ↓
Restart=on-failure
 ↓
systemd thử restart
```

---

## `systemctl is-active`

```bash
systemctl is-active myapp
```

Output thường:

```text
active
inactive
failed
```

Hữu ích cho script/check nhanh.

---

## `systemctl is-enabled`

```bash
systemctl is-enabled myapp
```

Dùng kiểm tra service có được cấu hình auto-start khi boot không.

Phân biệt:

```text
enabled ≠ running
enabled ≠ healthy
```

---

## `journalctl -u <service>`

```bash
sudo journalctl -u myapp
```

`-u` = chỉ log của systemd unit đó.

30 dòng gần nhất:

```bash
sudo journalctl -u myapp -n 30 --no-pager
```

Flags:

```text
-n 30      = lấy 30 log entries gần nhất
--no-pager = in thẳng output, không mở less
```

Realtime:

```bash
sudo journalctl -u myapp -f
```

`-f` = follow log tương tự `tail -f`.

Current boot:

```bash
sudo journalctl -u myapp -b
```

`-b` = log từ lần boot hiện tại.

---

# 9. File permission / process identity

## `ls -l`

```bash
ls -l /opt/myapp/config.json
```

Ví dụ:

```text
-rw-r----- 1 root appuser 120 config.json
```

Đọc:

```text
owner root    → rw-
group appuser → r--
other         → ---
```

---

## `namei -l`

```bash
namei -l /opt/myapp/config.json
```

Hữu ích khi file cuối có permission đúng nhưng process vẫn `Permission denied`.

Command này hiển thị permission của từng component trong path:

```text
/
/opt
/opt/myapp
/opt/myapp/config.json
```

Directory cha cũng cần quyền traverse (`x`).

---

## Chạy command bằng identity của service

```bash
sudo -u appuser cat /opt/myapp/config.json
```

Đây là cách verify rất mạnh:

> User thật của application có đọc file này được không?

Nếu trả:

```text
Permission denied
```

thì có evidence trực tiếp về permission problem.

---

# 10. Nginx / reverse proxy / upstream

## Check syntax

```bash
sudo nginx -t
```

Nếu OK:

```text
syntax is ok
test is successful
```

Dùng trước reload/restart sau khi sửa config.

---

## In toàn bộ effective config

```bash
sudo nginx -T
```

Khác `-t`:

```text
-t = validate syntax
-T = validate + dump toàn bộ config đã include
```

Tìm listener:

```bash
sudo nginx -T | grep -n 'listen'
```

Tìm upstream/proxy:

```bash
sudo nginx -T | grep -n 'proxy_pass'
```

---

## Test upstream trực tiếp

Nếu Nginx có:

```nginx
proxy_pass http://127.0.0.1:3000;
```

thì bypass Nginx:

```bash
curl -v http://127.0.0.1:3000/healthz
```

Nếu request qua Nginx trả `502` nhưng gọi upstream trực tiếp cũng `Connection refused`, suspect gần nhất là upstream process/listener.

---

## Nginx logs

Thông thường:

```bash
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
```

`access.log` giúp xác nhận request có tới Nginx không.

`error.log` hữu ích cho:

```text
connect() failed ... while connecting to upstream
upstream timed out
permission denied
certificate/config problems
```

Nếu Nginx chạy qua systemd cũng nên xem:

```bash
sudo journalctl -u nginx -n 50 --no-pager
```

---

# 11. Firewall trên Linux

Tool phụ thuộc distro/configuration.

## UFW

```bash
sudo ufw status verbose
```

Dùng xem rule đang allow/deny port nào.

Không thay đổi rule ngay khi chưa có evidence.

---

## nftables

```bash
sudo nft list ruleset
```

Trên Linux hiện đại, đây có thể là firewall rule set thực tế.

---

## iptables

Hệ thống cũ hoặc compatibility layer:

```bash
sudo iptables -L -n -v
```

Flags:

```text
-L = list rules
-n = không resolve IP/port thành name
-v = verbose counters/details
```

Firewall thường đáng kiểm tra sớm hơn khi symptom là timeout/drop, không phải khi application đã trả HTTP 500.

---

# 12. Packet-level debugging — dùng khi command phía trên chưa đủ

## `tcpdump`

Xem TCP traffic port 443:

```bash
sudo tcpdump -ni any tcp port 443
```

Flags:

```text
-n = không resolve IP/port thành name
-i any = capture mọi interface
```

Filter một host:

```bash
sudo tcpdump -ni any host 10.0.1.50
```

Quan sát handshake có thể giúp phân biệt:

```text
SYN đi ra nhưng không có reply
SYN → RST
SYN → SYN-ACK → ACK
```

Đây là tool mạnh nhưng nên dùng sau khi đã kiểm tra route/listener/service cơ bản.

---

# 13. Debug theo triệu chứng

## `Could not resolve host`

Ưu tiên:

```bash
dig +short <domain>
getent hosts <domain>
cat /etc/resolv.conf
resolvectl status
```

Layer suspect:

```text
DNS / name resolution
```

---

## `Connection refused`

Ưu tiên phía server:

```bash
ss -ltnp | grep :<port>
pgrep -af <process>
systemctl status <service>
journalctl -u <service> -n 50 --no-pager
```

Layer suspect:

```text
TCP / listener / process
```

---

## `Connection timed out`

Ưu tiên:

```bash
ip route get <destination-ip>
ping -c 4 <destination-ip>
tracepath <destination-ip>
```

Sau đó kiểm tra firewall/security rules.

Layer suspect:

```text
route / firewall / network path
```

---

## Certificate error

Ưu tiên:

```bash
curl -v https://<domain>
openssl s_client -connect <domain>:443 -servername <domain>
```

Layer suspect:

```text
TLS / certificate
```

---

## HTTP 500

Ưu tiên:

```bash
journalctl -u <app-service> -n 100 --no-pager
curl -v <dependency-health-endpoint>
```

Layer suspect:

```text
application / dependency
```

Không quay lại debug client firewall trước nếu client đã nhận HTTP response.

---

## HTTP 502

Ưu tiên:

```bash
sudo nginx -t
sudo nginx -T | grep -n 'proxy_pass'
curl -v http://<upstream-host>:<upstream-port>/healthz
ss -ltnp | grep :<upstream-port>
systemctl status <upstream-service>
journalctl -u <upstream-service> -n 50 --no-pager
```

Layer suspect:

```text
reverse proxy → upstream
```

---

# 14. Recommended debug flow

Khi nhận ticket kiểu:

```text
"API không vào được"
```

không bắt đầu bằng restart.

Đi theo evidence:

```text
1. Hostname resolve đúng không?
   dig / getent

2. Kernel sẽ route packet đi đâu?
   ip route get

3. TCP connect được không?
   curl -v / nc

4. Server có listener không?
   ss -ltnp

5. Nếu HTTPS, TLS có pass không?
   curl -v / openssl s_client

6. HTTP status là gì?
   curl -i

7. Reverse proxy có reach upstream không?
   curl upstream trực tiếp

8. Process/service có sống không?
   systemctl / pgrep

9. Log nói gì?
   journalctl / nginx error log

10. Sau khi có hypothesis mới fix.
```

Flow chuẩn:

```text
Observation
   ↓
Hypothesis
   ↓
Verification command
   ↓
Evidence
   ↓
Fix
   ↓
Verify again
```

Không nên:

```text
error
 ↓
restart ngẫu nhiên
 ↓
chmod 777
 ↓
tắt firewall
 ↓
thử lại
```

---

# 15. Command cheat sheet ngắn

```bash
# IP / interface
ip addr
ip link

# Route
ip route
ip route get 10.0.1.50

# DNS
dig api.example.com
dig +short api.example.com
getent hosts api.example.com
resolvectl status

# TCP / listener
ss -ltnp
ss -ltnp | grep :443
ss -tn
nc -vz 10.0.1.50 443

# HTTP / multi-layer
curl -v https://api.example.com/healthz
curl -i https://api.example.com/healthz
curl -I https://example.com
curl -sS -o /dev/null -w '%{http_code}\n' https://api.example.com/healthz

# DNS bypass
curl -v --resolve api.example.com:443:10.0.1.50 https://api.example.com/healthz

# TLS
openssl s_client -connect api.example.com:443 -servername api.example.com

# Process
pgrep -af node
ps aux | grep node

# systemd
systemctl status myapp --no-pager
systemctl is-active myapp
systemctl is-enabled myapp
sudo journalctl -u myapp -n 50 --no-pager
sudo journalctl -u myapp -f

# Permission
ls -l /opt/myapp/config.json
namei -l /opt/myapp/config.json
sudo -u appuser cat /opt/myapp/config.json

# Nginx
sudo nginx -t
sudo nginx -T
sudo tail -f /var/log/nginx/error.log

# Firewall
sudo ufw status verbose
sudo nft list ruleset

# Packet capture
sudo tcpdump -ni any tcp port 443
```

---

# 16. Điều cần nhớ hơn command

```text
dig success
≠ destination đúng

ping fail
≠ server chết

Trying IP:443
≠ TCP connected

Connection refused
≠ TCP established

process alive
≠ application healthy

enabled
≠ running

HTTP 500
→ client network path đã đi được tới HTTP layer

HTTP 502
→ reverse proxy nhận request nhưng upstream path có vấn đề
```

Mục tiêu cuối cùng không phải thuộc lệnh, mà là nhìn symptom và biết command nào tạo ra evidence cho layer tiếp theo.