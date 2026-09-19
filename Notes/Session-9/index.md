# Session 9 — Cache & Release Frontend

## Mục tiêu

Hiểu cache lifecycle và release lifecycle của một SPA frontend để tránh các lỗi kiểu:

```text
deploy thành công
+
browser/CDN vẫn dùng entrypoint cũ
+
asset version cũ đã biến mất
=
broken frontend
```

Checkpoint của buổi:

```text
index.html
→ revalidate

runtime-config.json
→ không stale

hashed JS/CSS
→ long cache + immutable

missing asset
→ 404, không SPA fallback

release
→ assets first, HTML last

old assets
→ giữ đủ lâu cho active users + rollback

debug stale release
→ browser / service worker / CDN / origin
```

---

## 1. Cache matrix

Frontend resources có lifecycle khác nhau nên không dùng cùng một cache policy.

| Resource | Cache-Control | Ý nghĩa |
| --- | --- | --- |
| `index.html` | `no-cache` | Có thể lưu nhưng phải revalidate trước khi reuse |
| `runtime-config.json` | `no-store` | Không nên lưu; luôn lấy runtime config hiện tại |
| `/assets/*.js`, `/assets/*.css` có hash | `public, max-age=31536000, immutable` | Cache lâu vì content đổi thì URL đổi |

Mental model:

```text
index.html
→ mutable entrypoint

runtime-config.json
→ mutable runtime input

hashed assets
→ immutable content-addressed/versioned files
```

---

## 2. `no-cache` khác `no-store`

### `no-cache`

Tên dễ gây hiểu nhầm. Nó không có nghĩa là "không được cache".

```text
store allowed
+
validate before reuse
```

Client có thể giữ `index.html`, nhưng trước khi dùng lại phải hỏi server xem version có còn hợp lệ không.

### `no-store`

```text
do not store this response
```

Phù hợp với `runtime-config.json` trong lab vì runtime config có thể đổi độc lập với frontend image.

---

## 3. ETag và HTTP 304 revalidation

Server có thể trả:

```http
ETag: "6aab4384-1ca"
```

Client gửi conditional request:

```http
If-None-Match: "6aab4384-1ca"
```

Lab xác nhận Nginx trả:

```text
HTTP/1.1 304 Not Modified
ETag: "6aab4384-1ca"
```

Flow:

```text
client có cached index.html
        ↓
If-None-Match: old-etag
        ↓
server kiểm tra
        ↓
file chưa thay đổi
        ↓
304 Not Modified
        ↓
client reuse cached body
```

Command:

```bash
curl -I http://localhost:8080/index.html

curl -i \
  -H 'If-None-Match: "<ETAG>"' \
  http://localhost:8080/index.html
```

`Last-Modified` / `If-Modified-Since` cũng có thể dùng để revalidate, nhưng ETag là validator rõ ràng hơn cho resource version.

---

## 4. Hashed assets và long-term cache

Vite tạo asset kiểu:

```text
/assets/index-DsRdjuXJ.js
/assets/index-3zoZfb4g.css
```

Khi source thay đổi, bundle content thay đổi và hash trong filename cũng thay đổi:

```text
Release A
index-AAA.js

Release B
index-BBB.js
```

Do đó:

```text
same URL
→ same content

new content
→ new URL
```

Cho nên hashed asset có thể dùng:

```http
Cache-Control: public, max-age=31536000, immutable
```

Asset cũ còn nằm trong browser cache không gây vấn đề nếu HTML mới không reference nó nữa.

---

## 5. Bug ban đầu — missing asset bị SPA fallback thành `index.html`

Config ban đầu:

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

Lab:

```bash
curl -i http://localhost:8080/assets/this-file-does-not-exist.js
```

Trước khi fix, kết quả:

```text
HTTP/1.1 200 OK
Content-Type: text/html
```

Body thực tế là `index.html`.

Flow sai:

```text
GET /assets/missing.js
        ↓
file không tồn tại
        ↓
SPA fallback
        ↓
index.html
        ↓
200 + text/html
```

Browser đang mong JavaScript nhưng nhận HTML, có thể sinh lỗi MIME/module gây hiểu nhầm.

Root cause thật:

```text
asset missing
```

---

## 6. Tách asset routing khỏi SPA fallback

Config sau khi sửa:

```nginx
server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    location = /runtime-config.json {
        add_header Cache-Control "no-store";
        try_files $uri =404;
    }

    location = /index.html {
        add_header Cache-Control "no-cache";
        try_files $uri =404;
    }

    location /assets/ {
        add_header Cache-Control "public, max-age=31536000, immutable";
        try_files $uri =404;
    }

    location /api/ {
        proxy_pass http://api:4000/;

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

Hai behavior phải khác nhau:

```text
SPA route:
/dashboard/users
→ physical file không tồn tại
→ index.html
→ React Router xử lý

Static asset:
/assets/old.js
→ file không tồn tại
→ 404
```

Checkpoint đã xác nhận:

```text
SPA route missing
→ fallback index.html ✅

asset missing
→ HTTP 404 ✅
```

Sau khi sửa `nginx.conf`, do file được COPY vào Docker image nên cần rebuild frontend:

```bash
docker compose up -d --build frontend
```

---

## 7. Safe frontend release — assets first, HTML last

Giả sử:

```text
Release A:
index.html → AAA.js

Release B:
index.html → BBB.js
```

Deployment không an toàn:

```text
delete A
   ↓
publish index.html B
   ↓
upload BBB.js
```

Có khoảng thời gian HTML đã reference file chưa tồn tại.

Safe flow:

```text
1. Build Release B
        ↓
2. Upload assets B
        ↓
3. Verify assets B = 200
        ↓
4. Publish index.html B
        ↓
5. Observe release
        ↓
6. Cleanup old assets later
```

Nguyên tắc:

```text
ASSETS FIRST
HTML LAST
```

`index.html` là mutable entrypoint quyết định browser sẽ request asset version nào. Vì vậy asset mới phải sẵn sàng trước khi HTML mới được publish.

---

## 8. Lazy loading và lý do cần old asset retention

Ví dụ app có dynamic import / lazy route:

```ts
const Settings = lazy(() => import("./pages/Settings"));
```

User mở Release A trước khi deploy nhưng chưa vào Settings.

Sau deploy B:

```text
user đang chạy Release A
        ↓
click Settings
        ↓
request Settings-OLDHASH.js
```

Nếu old asset đã bị xóa:

```text
GET /assets/Settings-OLDHASH.js
→ 404
```

Root cause trực tiếp ở HTTP layer:

```text
HTTP 404
```

Application symptom có thể là dynamic import failure, ChunkLoadError hoặc blank screen.

---

## 9. Cache retention khác origin asset retention

Hai khái niệm độc lập:

```text
CACHE RETENTION
"browser/CDN giữ COPY của file bao lâu?"

                !=

ORIGIN RETENTION
"server/storage giữ ORIGINAL file bao lâu?"
```

Ví dụ:

```http
Cache-Control: public, max-age=31536000, immutable
```

không có nghĩa server sẽ giữ file 1 năm.

Nó chỉ điều khiển cache behavior của client/intermediate cache.

---

## 10. Old asset retention — khi nào xóa và config ở đâu?

Không có một retention window bắt buộc.

Policy thường dựa trên:

```text
longest active user session
+
rollback window
+
deployment frequency
```

Ví dụ:

```text
keep last 3–5 releases

hoặc

keep assets for 7 / 14 / 30 days
```

Old assets thường được cleanup bởi deployment/storage system, không phải Nginx.

### Object storage + CDN

Ví dụ:

```text
Browser
   ↓
CDN
   ↓
S3/Object Storage
```

Storage có thể đồng thời chứa:

```text
/assets/
├── AAA.js
├── BBB.js
├── CCC.js
└── DDD.js
```

Mỗi release upload thêm hashed assets mới, không overwrite asset cũ.

Cleanup được cấu hình bằng storage lifecycle rule, ví dụ:

```text
prefix: assets/
expiration: 30 days
```

### CI/CD cleanup

Có thể có scheduled job:

```bash
find /var/www/assets \
  -type f \
  -mtime +7 \
  -delete
```

Trong production nên cân nhắc giữ theo release/version thay vì chỉ dựa vào timestamp.

### Versioned release directories

```text
/var/www/releases/
├── release-A/
├── release-B/
└── release-C/

current → release-C
```

Rollback có thể đổi symlink/reference về release cũ, còn cleanup job giữ last N releases.

---

## 11. Limitation của `portal-saas` hiện tại

Frontend Dockerfile hiện copy đúng `dist` của release hiện tại:

```dockerfile
COPY --from=build \
    /app/apps/frontend/dist \
    /usr/share/nginx/html
```

Do đó:

```text
image A
→ assets A

image B
→ assets B
```

Khi container A bị replace bởi container B, Nginx mới chỉ thấy filesystem của image B.

Hiện tại:

```text
browser hashed-asset cache:
1 year ✅

origin old-asset retention:
none ❌
```

Đây là limitation chấp nhận được cho lab hiện tại và sẽ nối sang bài artifact/release/rollback.

---

## 12. Old assets giúp rollback

Nếu storage đang giữ:

```text
AAA.js
BBB.js
```

và Release B có bug:

```text
index.html → BBB.js
```

có thể rollback entrypoint về:

```text
index.html → AAA.js
```

vì AAA.js vẫn tồn tại.

Nếu deploy B đã xóa assets A:

```text
rollback index.html A
        ↓
request AAA.js
        ↓
404
```

Do đó old asset retention hỗ trợ cả:

```text
active users
+
lazy chunks
+
rollback
```

---

## 13. Cache layers cần phân biệt

Production request path có thể là:

```text
Browser HTTP Cache
        ↓
Service Worker
        ↓
Proxy / CDN
        ↓
Load Balancer
        ↓
Nginx origin
```

Khi user báo "deploy mới rồi nhưng vẫn thấy code cũ", không nên restart random server trước.

Cần xác định response đang đến từ layer nào.

### Browser cache

DevTools → Network có thể hiện:

```text
from memory cache
from disk cache
```

### Service Worker

Service worker có thể intercept request và trả resource từ Cache Storage trước khi network request tới Nginx.

Debug:

```text
DevTools
→ Application
→ Service Workers

DevTools
→ Application
→ Cache Storage
```

Nếu:

```text
curl public domain → Release B
Chrome             → Release A
```

hãy nghi browser cache / service worker trước.

### CDN / proxy cache

Headers thường hữu ích:

```text
Age
Via
X-Cache
CF-Cache-Status
```

Nếu:

```text
origin → B
public CDN → A
```

thì vấn đề nằm ở CDN/proxy layer.

---

## 14. CDN stale HTML — recovery và prevention

Incident:

```text
Origin Nginx → Release B
CDN          → Release A
User         → Release A
```

Debug:

```bash
curl -I https://app.example.com/index.html
curl -i https://app.example.com/index.html
```

Tìm các dấu hiệu cache hit như:

```text
Age: ...
X-Cache: HIT
CF-Cache-Status: HIT
```

### Recovery

Targeted purge/invalidate:

```text
/index.html
/
```

Không nên mặc định purge toàn bộ hashed assets.

Sau purge:

```text
CDN cache entry gone
        ↓
request tiếp theo
        ↓
origin
        ↓
index.html B
```

### Prevention

CDN phải tôn trọng lifecycle khác nhau:

```text
/index.html
→ short TTL / revalidate
→ no-cache semantics

/runtime-config.json
→ bypass / no-store

/assets/*
→ long cache
→ immutable
```

Purge là cách recover incident; cache policy đúng mới là cách ngăn incident lặp lại.

---

## 15. Rolling deployment failure mode

Giả sử hai frontend replicas đang chạy song song:

```text
frontend-1 → Release A
frontend-2 → Release B
```

Load balancer có thể tạo flow:

```text
GET /
 ↓
frontend-2
 ↓
index.html B
 ↓
GET BBB.js
 ↓
frontend-1
 ↓
404
```

Đây là lý do static hashed assets thường phù hợp với shared storage/CDN hoặc release strategy đảm bảo asset compatibility trong thời gian rollout.

---

## 16. Debugging flow cho stale/broken frontend release

Không restart ngẫu nhiên. Đi theo evidence:

```text
1. Artifact/image hiện tại đúng chưa?
        ↓
2. Origin Nginx trả release nào?
        ↓
3. Public domain/CDN trả release nào?
        ↓
4. Browser nhận release nào?
        ↓
5. Service Worker có intercept không?
        ↓
6. Asset được HTML reference có HTTP 200 không?
```

Một số phân loại nhanh:

```text
origin B, CDN A
→ CDN cache problem

curl B, Chrome A
→ browser cache / service worker

HTML B, BBB.js = 404
→ release ordering / upload asset problem

missing JS returns 200 text/html
→ SPA fallback misconfiguration

missing JS returns 404
→ routing behavior đúng; tiếp tục tìm tại release compatibility
```

---

# Command reference — Session 9

```bash
# inspect HTML/cache headers
curl -I http://localhost:8080/index.html

# inspect runtime config
curl -I http://localhost:8080/runtime-config.json

# inspect actual HTML
curl -s http://localhost:8080/index.html

# missing asset must return 404
curl -i http://localhost:8080/assets/this-file-does-not-exist.js

# actual hashed asset
curl -I http://localhost:8080/assets/<HASHED_FILE>.js

# conditional request / ETag revalidation
curl -i \
  -H 'If-None-Match: "<ETAG>"' \
  http://localhost:8080/index.html

# verbose request/response
curl -v http://localhost:8080/index.html -o /dev/null

# rebuild frontend after nginx.conf changes
docker compose up -d --build frontend
```

---

# Checkpoint Session 9

Đã đạt:

- Hiểu lifecycle khác nhau của HTML, runtime config và hashed assets.
- Phân biệt `no-cache`, `no-store`, `max-age` và `immutable`.
- Dùng ETag để quan sát revalidation và HTTP `304 Not Modified`.
- Chứng minh missing JS asset trước fix bị trả `200 text/html`.
- Tách `/assets/` khỏi SPA fallback để missing asset trả `404`.
- Giữ SPA deep route fallback về `index.html`.
- Hiểu content hash giúp asset cache lâu mà vẫn release version mới.
- Hiểu safe release: **assets first, HTML last**.
- Hiểu lazy loading có thể request chunk cũ sau khi deploy release mới.
- Phân biệt cache retention và origin asset retention.
- Hiểu old asset retention hỗ trợ active sessions và rollback.
- Biết old asset thường được cleanup bởi storage lifecycle / CI/CD / release policy, không phải Cache-Control.
- Xác định `portal-saas` hiện tại chưa có origin old-asset retention vì mỗi Docker image chỉ chứa `dist` của release hiện tại.
- Phân biệt browser cache, service worker, CDN/proxy cache và origin.
- Biết debug stale HTML bằng `Age`, `X-Cache`, `CF-Cache-Status`, `Via`.
- Hiểu targeted CDN purge/invalidation là recovery mechanism, không thay thế cache policy đúng.
- Hiểu rolling deployment có thể gây cross-version asset 404 nếu replicas không chia sẻ static assets.

## Mental model cuối buổi

```text
                    FRONTEND RELEASE

Build
  ↓
hashed assets
  ↓
upload assets
  ↓
verify assets = 200
  ↓
publish mutable index.html
  ↓
user receives new HTML
  ↓
HTML references new hashed assets
  ↓
retain old assets for active users + rollback
  ↓
cleanup old versions later
```

Cache policy:

```text
index.html
→ no-cache
→ revalidate before reuse

runtime-config.json
→ no-store
→ avoid stale runtime config

/assets/*
→ public
→ max-age=31536000
→ immutable
```

## Next — Session 10: Artifact & Rollback

Nối tiếp trực tiếp từ Session 9:

```text
release version
+
artifact/image identity
+
runtime config compatibility
+
retention
+
rollback
```

Mục tiêu tiếp theo là hiểu một release cần được truy vết bằng artifact/version nào, giữ artifact cũ bao lâu, rollback về release nào và kiểm tra compatibility giữa image, static assets và runtime config.
