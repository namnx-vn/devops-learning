# Session 4 — Nginx, Reverse Proxy & Browser

## Mục tiêu buổi học

- Hiểu vai trò của Nginx khi đứng trước SPA và Node API.
- Phân biệt URL mà browser truy cập với địa chỉ nội bộ mà reverse proxy dùng để gọi upstream.
- Serve frontend SPA bằng Nginx.
- Hiểu cách `try_files` giải quyết SPA deep route như `/dashboard`.
- Reverse proxy `/api/` tới Node API chạy ở port `3000`.
- Hiểu cách Nginx chọn `location` cụ thể hơn.
- Hiểu ảnh hưởng của dấu `/` cuối trong `proxy_pass`.
- Hiểu `502 Bad Gateway` và cách debug theo evidence.
- Phân biệt lỗi CORS với lỗi reverse proxy/upstream.
- Hiểu same-origin, `Secure`, `SameSite`, `HttpOnly` và forwarded headers.
- Debug một case browser load được SPA nhưng application JavaScript crash.

Mental model xuyên suốt:

```text
Browser
   |
   | HTTP/HTTPS
   v
Nginx :80/:443
   |
   |-- /, /dashboard, /assets/* -> SPA static files
   |
   `-- /api/*
         |
         | internal HTTP
         v
      Node :3000
```

Nguyên tắc chính:

> Browser không cần biết Node đang chạy ở port nội bộ nào. Browser nói chuyện với public entry point; Nginx chịu trách nhiệm route request tới static frontend hoặc upstream application.

---

## 1. `127.0.0.1` luôn thuộc máy đang thực hiện connection

Nếu browser chạy trên Mac và mở:

```text
http://127.0.0.1:3000/healthz
```

thì browser đang tìm port `3000` trên chính Mac, không phải Ubuntu VM.

```text
Browser trên Mac
      |
      | 127.0.0.1:3000
      v
Mac localhost
```

Muốn truy cập Node trực tiếp trên VM phải dùng IP VM nếu Node được bind/expose phù hợp:

```text
http://<VM-IP>:3000/healthz
```

Trong lab Node listen:

```js
server.listen(PORT, "0.0.0.0")
```

nên process có thể nhận connection qua các interface của VM nếu firewall/network cho phép.

Điểm cần nhớ:

```text
127.0.0.1
= localhost của process/client đang dùng địa chỉ đó
```

---

## 2. Node API baseline

Node API từ Session 2 chạy tại:

```text
:3000
```

Endpoint ban đầu:

```text
/healthz
```

Trong lúc kiểm tra:

```bash
curl -i http://127.0.0.1:3000/version
```

nhận:

```text
HTTP/1.1 404 Not Found
```

Root cause không phải network. Source lúc đó chỉ có `/healthz`, nên `/version` rơi xuống nhánh `404` của application.

Mental model:

```text
TCP connect được       ✅
HTTP response nhận được ✅
HTTP 404               ✅

=> network path hoạt động
=> application route không tồn tại
```

Có thể thêm endpoint:

```js
if (req.url === "/version") {
  return res.end(
    JSON.stringify({
      version: "1.0.0",
    })
  );
}
```

Sau khi sửa service:

```bash
sudo systemctl restart myapp
curl -i http://127.0.0.1:3000/version
```

---

## 3. Nginx làm Reverse Proxy

Config cơ bản:

```nginx
server {
    listen 80;
    server_name _;

    location /api/ {
        proxy_pass http://127.0.0.1:3000/;
    }
}
```

Browser gọi:

```text
http://<VM-IP>/api/healthz
```

Flow:

```text
Browser
   |
   | GET /api/healthz
   v
Nginx :80
   |
   | proxy_pass
   v
Node :3000
   |
   | GET /healthz
   v
200 OK
```

Browser chỉ thấy public URL:

```text
http://<VM-IP>/api/healthz
```

Nginx mới biết upstream nội bộ:

```text
http://127.0.0.1:3000
```

---

## 4. Dấu `/` trong `proxy_pass`

Case:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:3000/;
}
```

Request:

```text
/api/healthz
```

Nginx thay prefix `/api/` bằng `/`, nên upstream nhận:

```text
/healthz
```

```text
/api/healthz
     |
     v
/healthz
```

Nếu viết:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:3000;
}
```

không có `/` cuối ở `proxy_pass`, upstream có thể nhận URI:

```text
/api/healthz
```

Với Node chỉ implement `/healthz`, kết quả có thể là `404`.

Điểm cần nhớ:

> Khi debug reverse proxy, không chỉ kiểm tra host/port upstream; phải kiểm tra URI mà upstream thực sự nhận được.

---

## 5. Luôn test Nginx config trước khi reload

Sau khi sửa config:

```bash
sudo nginx -t
```

Expected:

```text
syntax is ok
test is successful
```

Sau đó mới:

```bash
sudo systemctl reload nginx
```

Flow an toàn:

```text
edit config
   |
   v
nginx -t
   |
   | success
   v
reload nginx
```

`reload` phù hợp hơn `restart` khi chỉ cần nạp config mới và muốn hạn chế gián đoạn service.

---

## 6. `502 Bad Gateway`

Nếu:

```bash
curl -i http://127.0.0.1:3000/healthz
```

trả `200`, nhưng:

```bash
curl -i http://127.0.0.1/api/healthz
```

trả:

```text
502 Bad Gateway
```

thì evidence cho thấy:

```text
Client -> Nginx        ✅
Node trực tiếp :3000   ✅
Nginx -> upstream      ❌
```

Một incident điển hình:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:3001/;
}
```

trong khi Node thật chạy ở `3000`.

Debug:

```bash
curl -i http://127.0.0.1:3000/healthz
ss -ltnp | grep :300
sudo tail -n 30 /var/log/nginx/error.log
```

Log có thể có dạng:

```text
connect() failed (111: Connection refused) while connecting to upstream
```

Root cause chain:

```text
502
 |
 v
Nginx không gọi được upstream
 |
 v
upstream host/port/process sai hoặc unavailable
```

Fix ví dụ:

```nginx
proxy_pass http://127.0.0.1:3000/;
```

sau đó:

```bash
sudo nginx -t
sudo systemctl reload nginx
curl -i http://127.0.0.1/api/healthz
```

---

## 7. Vì sao CORS không sửa được `502`

CORS là browser security policy dành cho cross-origin request.

Nó kiểm soát JavaScript trong browser có được phép đọc response từ origin khác hay không.

Ví dụ:

```text
Frontend: http://frontend.example.com
API:      https://api.example.com
```

Server có thể trả:

```http
Access-Control-Allow-Origin: http://frontend.example.com
```

Nhưng `502` là lỗi server-side giữa reverse proxy và upstream:

```text
Browser/curl
    |
    v
Nginx ✅
    |
    v
Node upstream ❌
    |
    v
502
```

Thêm CORS header không thể làm port sai thành đúng hoặc làm process đã chết sống lại.

Một evidence mạnh:

```bash
curl http://127.0.0.1/api/healthz
```

nếu cũng trả `502`, thì không thể coi CORS là root cause vì `curl` không enforce CORS giống browser.

Mental model:

```text
502        -> reverse proxy / upstream
CORS error -> browser cross-origin policy
```

---

## 8. Same Origin

Origin được xác định bởi:

```text
scheme + host + port
```

Ví dụ:

```text
http://10.0.0.20/
http://10.0.0.20/api/healthz
```

là same origin vì cùng:

```text
scheme = http
host   = 10.0.0.20
port   = 80
```

Path khác nhau không làm origin khác nhau.

Các ví dụ khác origin:

```text
http://10.0.0.20
http://10.0.0.20:3000   # khác port

http://10.0.0.20
https://10.0.0.20       # khác scheme

http://app.example.com
http://api.example.com   # khác host
```

Một lợi ích của reverse proxy `/api/` là frontend và API có thể cùng public origin, giảm nhu cầu CORS cho flow thông thường.

---

## 9. SPA Deep Route và `try_files`

React Router quản lý route client-side như:

```text
/dashboard
```

Khi user click từ `/` sang `/dashboard`, React Router có thể đổi URL và render component mà không reload document từ server.

Nhưng khi refresh trực tiếp:

```text
GET /dashboard
```

Nginx nhận request thật.

Nếu Nginx tìm file:

```text
/var/www/myapp/dashboard
```

và file không tồn tại thì có thể trả `404`.

Config SPA fallback:

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

Flow:

```text
GET /dashboard
       |
       v
Nginx tìm $uri
       |
       | không có file
       v
/index.html
       |
       v
React load
       |
       v
React Router đọc /dashboard
       |
       v
Dashboard component
```

Điểm quan trọng:

> Nginx không tự động biết `/dashboard` là React route. Ta phải cấu hình fallback về `index.html`.

---

## 10. Nginx `location` cụ thể hơn được ưu tiên

Với:

```nginx
location / {
    try_files $uri $uri/ /index.html;
}

location /api/ {
    proxy_pass http://127.0.0.1:3000/;
}
```

request:

```text
/api/healthz
```

match cả `/` và `/api/`, nhưng prefix `/api/` cụ thể hơn nên được chọn.

```text
/api/healthz
 |
 |-- /       match
 `-- /api/   match + cụ thể hơn -> chọn
```

Điều này giúp `/api/*` không bị SPA fallback trả `index.html`.

API path sai nên trả lỗi API thật, ví dụ JSON `404`, thay vì HTML của frontend.

---

## 11. Build SPA và copy vào Nginx

Với React/Vite:

```bash
npm run build
```

thường tạo:

```text
dist/
├── index.html
└── assets/
```

Từ Mac, copy build vào Multipass VM:

```bash
multipass transfer --recursive \
  /Users/tt/my_project/portal-saas/apps/frontend/dist \
  devops-lab:/tmp/dist
```

Trong Ubuntu VM:

```bash
sudo mkdir -p /var/www/myapp
sudo cp -r /tmp/dist/* /var/www/myapp/
ls -la /var/www/myapp
```

Mục tiêu:

```text
/var/www/myapp/
├── index.html
└── assets/
```

Không nên để thành:

```text
/var/www/myapp/dist/index.html
```

nếu Nginx đang dùng:

```nginx
root /var/www/myapp;
```

---

## 12. Nginx config hoàn chỉnh của lab

```nginx
server {
    listen 80;
    server_name _;

    root /var/www/myapp;
    index index.html;

    location /api/ {
        proxy_pass http://127.0.0.1:3000/;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

Verify:

```bash
sudo nginx -t
sudo systemctl reload nginx

curl -I http://127.0.0.1/
curl -I http://127.0.0.1/dashboard
curl -i http://127.0.0.1/api/healthz
```

Expected:

```text
/                     -> 200
/dashboard            -> 200 + SPA fallback
/api/healthz          -> 200 từ Node qua Nginx
```

---

## 13. Forwarded Headers

Reverse proxy tạo hai connection khác nhau:

```text
Browser -> Nginx
Nginx   -> Node
```

Node có thể chỉ nhìn thấy Nginx là peer trực tiếp.

Các header thường dùng:

```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

Ý nghĩa:

```text
Host
-> hostname client dùng

X-Real-IP
-> IP client mà Nginx nhìn thấy

X-Forwarded-For
-> chain IP qua proxy

X-Forwarded-Proto
-> protocol gốc của request, http hoặc https
```

Case quan trọng:

```text
Browser --HTTPS--> Nginx --HTTP--> Node
```

Nếu Node chỉ nhìn connection trực tiếp, nó thấy HTTP.

Header:

```http
X-Forwarded-Proto: https
```

cho backend biết browser ban đầu dùng HTTPS.

---

## 14. Cookie: `Secure`, `HttpOnly`, `SameSite`

Ví dụ:

```http
Set-Cookie: session=abc123; Secure; HttpOnly; SameSite=Lax
```

Ý nghĩa:

```text
Secure
-> browser chỉ gửi cookie qua HTTPS

HttpOnly
-> JavaScript không đọc được cookie qua document.cookie

SameSite
-> kiểm soát việc gửi cookie trong request liên quan tới site khác
```

CORS và cookie policy là hai cơ chế khác nhau:

```text
CORS
-> JavaScript cross-origin có được đọc response không

Cookie attributes
-> browser có gửi/lộ cookie trong context nào không
```

---

## 15. Browser load được SPA nhưng application vẫn có thể crash

Sau khi deploy frontend, browser truy cập:

```text
http://192.168.252.2/dashboard
```

Nginx đã trả được SPA, nhưng DevTools Console báo lỗi kiểu:

```text
BrowserAuthError: crypto_nonexistent
```

Bundle JavaScript đã được tải và execute, nên đây không còn là lỗi `try_files` hay Nginx static file routing.

Evidence chain:

```text
IP reachable          ✅
TCP :80               ✅
HTTP                  ✅
Nginx                 ✅
/dashboard fallback   ✅
index.html            ✅
JS bundle loaded      ✅
Application JS        ❌
```

Frontend sử dụng MSAL/browser crypto và page được truy cập qua HTTP trên IP private, Chrome hiển thị `Not Secure`.

Đây là ví dụ quan trọng của nguyên tắc Session 3:

> Layer nào đã có evidence thành công thì không quay lại sửa ngẫu nhiên.

Một cách lab thuận tiện là SSH tunnel từ Mac:

```bash
ssh -L 8080:127.0.0.1:80 ubuntu@192.168.252.2
```

sau đó truy cập:

```text
http://localhost:8080/dashboard
```

Kiến trúc production-like hơn là terminate HTTPS tại Nginx:

```text
Browser --HTTPS :443--> Nginx --HTTP :3000--> Node
```

---

# Debug Checklist — Session 4

Khi frontend/API có vấn đề, kiểm tra theo thứ tự evidence:

```bash
# Nginx service
sudo systemctl status nginx --no-pager

# Nginx listener
ss -ltnp | grep :80

# Syntax config
sudo nginx -t

# Static SPA
curl -I http://127.0.0.1/
curl -I http://127.0.0.1/dashboard

# Node trực tiếp
curl -i http://127.0.0.1:3000/healthz

# Node qua reverse proxy
curl -i http://127.0.0.1/api/healthz

# Nginx error log
sudo tail -n 50 /var/log/nginx/error.log

# Node listener
ss -ltnp | grep :3000

# Node service/log
sudo systemctl status myapp --no-pager
sudo journalctl -u myapp -n 50 --no-pager
```

Mental model:

```text
Browser không mở được host
-> network / listener

Nginx 404 ở /dashboard
-> static root / try_files

/api trả 502
-> Nginx -> upstream

/api trả 404 JSON
-> upstream đã nhận request, kiểm tra path/application route

Browser báo CORS
-> origin + response headers

HTML/JS đã load nhưng màn hình trắng
-> kiểm tra DevTools Console/Network, application runtime
```

---

# Checkpoint Buổi 4

Sau buổi học cần tự giải thích được:

```text
1. Browser dùng URL public nào?
2. Nginx route request theo `location` nào?
3. Khi nào request được serve static và khi nào proxy tới Node?
4. Vì sao refresh `/dashboard` cần `try_files`?
5. Dấu `/` trong `proxy_pass` ảnh hưởng URI upstream thế nào?
6. Tại sao `502` không phải lỗi CORS?
7. Same origin được xác định bởi những thành phần nào?
8. Vì sao backend sau reverse proxy cần forwarded headers?
9. `Secure`, `HttpOnly`, `SameSite` giải quyết vấn đề gì?
10. Làm sao chứng minh lỗi nằm ở Nginx, upstream hay application browser?
```

Checkpoint thực hành:

```text
SPA root                 -> hoạt động
SPA deep route           -> Nginx fallback index.html
/api/healthz             -> đi qua Nginx tới Node
502 incident             -> biết debug upstream bằng log/listener/direct curl
CORS vs 502              -> phân biệt được
Browser runtime incident -> khoanh vùng được tới application JS
```

---

# Tổng kết Mental Model

```text
Browser
   |
   | public URL
   v
Nginx
   |
   |-- static request
   |      |
   |      `-> /var/www/myapp
   |             |
   |             `-> index.html / assets
   |
   `-- /api/*
          |
          `-> proxy_pass
                 |
                 v
             Node :3000
```

Khi debug:

```text
đừng đoán
   |
   v
xác định request đã đi tới layer nào
   |
   v
thu evidence
   |
   v
chỉ debug layer đầu tiên bị fail
```

Buổi tiếp theo theo plan:

```text
Session 5 — Bash & Checkpoint tuần 1
```

Mục tiêu tiếp theo là viết smoke-test script kiểm tra HTTP status, version, timeout và exit code, sau đó dùng script để xác nhận service fail/recover tự động.
