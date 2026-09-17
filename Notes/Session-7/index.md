# Session 7 — Config & Secrets

## Mục tiêu

Phân biệt rõ ba loại giá trị:

```text
build-time metadata
vs
runtime public config
vs
secret
```

Checkpoint của buổi:

```text
same frontend artifact / image
        +
runtime-config A / B
        ↓
chạy được nhiều environment
```

và chứng minh frontend bundle/image không chứa credential.

---

## 1. Mental model

```text
BUILD-TIME
"Artifact này là bản nào?"

Git SHA
version
build metadata


RUNTIME PUBLIC CONFIG
"Artifact này chạy ở đâu / như thế nào?"

API URL
environment name
public OAuth/Entra client ID
tenant ID
redirect URI
non-sensitive feature flags


SECRET
"Trusted system đang sở hữu credential/quyền gì?"

DB password
JWT signing key
AWS secret key
private token
OAuth client secret
```

Nguyên tắc quan trọng:

> Nếu browser cần giá trị để hoạt động thì phải giả định user/attacker cũng đọc được giá trị đó.

---

## 2. Build-time config với Vite

Ban đầu frontend dùng:

```ts
import.meta.env.VITE_ENTRA_CLIENT_ID
import.meta.env.VITE_ENTRA_TENANT_NAME
import.meta.env.VITE_ENTRA_TENANT_ID
import.meta.env.VITE_REDIRECT_URI
```

Khi chạy:

```bash
VITE_ENTRA_CLIENT_ID=session7-dev-client \
VITE_ENTRA_TENANT_NAME=session7-dev \
VITE_ENTRA_TENANT_ID=session7-dev-tenant \
VITE_REDIRECT_URI=http://localhost:8081 \
pnpm --filter frontend build
```

Vite thay giá trị vào JavaScript bundle lúc build.

Bằng chứng:

```bash
grep -R "session7-dev-client" apps/frontend/dist
```

=> giá trị xuất hiện trong bundle.

Build lại với giá trị PROD cho checksum artifact khác DEV.

Mental model:

```text
DEV config + source
      ↓
    build
      ↓
DEV artifact

PROD config + same source
      ↓
    build
      ↓
PROD artifact
```

=> environment-specific build làm mất tính "build once, promote many".

---

## 3. Runtime public config

Frontend được đổi sang load:

```text
/runtime-config.json
```

Ví dụ:

```json
{
  "schemaVersion": 1,
  "environment": "dev",
  "apiBaseUrl": "https://api-dev.example.com",
  "entra": {
    "clientId": "dev-client",
    "tenantName": "devtenant",
    "tenantId": "dev-tenant-id",
    "redirectUri": "http://localhost:8081"
  }
}
```

Flow bootstrap:

```text
index.html
    ↓
main.tsx
    ↓
fetch /runtime-config.json
    ↓
validate config
    ↓
create MSAL config
    ↓
initialize MSAL
    ↓
render React
```

Frontend image chỉ build một lần. Hai container dùng cùng image nhưng mount hai runtime config khác nhau.

```text
              SAME IMAGE
                  |
           +------+------+
           |             |
           v             v
        DEV config    PROD config
```

---

## 4. Checkpoint: same image, different environment

Hai container:

```bash
docker run -d \
  --name portal-dev \
  -p 8081:80 \
  -v "$PWD/runtime-configs/dev.json:/usr/share/nginx/html/runtime-config.json:ro" \
  portal-frontend:$SHORT_SHA
```

```bash
docker run -d \
  --name portal-prod \
  -p 8082:80 \
  -v "$PWD/runtime-configs/prod.json:/usr/share/nginx/html/runtime-config.json:ro" \
  portal-frontend:$SHORT_SHA
```

Kiểm tra cùng image:

```bash
docker inspect portal-dev --format 'DEV={{.Image}}'
docker inspect portal-prod --format 'PROD={{.Image}}'
```

Image SHA giống nhau.

Kiểm tra checksum bundle:

```bash
docker exec portal-dev sh -c '
cd /usr/share/nginx/html &&
find assets -type f -exec sha256sum {} \; |
sort |
sha256sum
'
```

```bash
docker exec portal-prod sh -c '
cd /usr/share/nginx/html &&
find assets -type f -exec sha256sum {} \; |
sort |
sha256sum
'
```

Checksum assets giống nhau trong khi `runtime-config.json` khác nhau.

=> đạt mục tiêu:

```text
same bundle
+
different runtime public config
=
different environments without rebuild
```

---

## 5. `.env` không tự động là secret

Ví dụ nguy hiểm:

```env
VITE_FAKE_DATABASE_PASSWORD=SESSION7_DB_SECRET_987654
```

Nếu frontend code dùng biến này và chạy Vite build, giá trị sẽ được nhúng vào JS bundle.

```text
.env
 ↓
Vite build
 ↓
JavaScript bundle
 ↓
browser
```

Có thể chứng minh bằng:

```bash
grep -R "SESSION7_DB_SECRET_987654" apps/frontend/dist
```

Bài học:

> `.env` chỉ là một cơ chế truyền configuration, không phải security boundary.

Đặc biệt với Vite, biến `VITE_*` được thiết kế để frontend code có thể sử dụng và do đó phải coi là public.

---

## 6. Secret không được tồn tại ở frontend

Không đưa các giá trị sau vào frontend bundle hoặc runtime config:

```text
DATABASE_PASSWORD
AWS_SECRET_ACCESS_KEY
JWT_SIGNING_SECRET
private API token
OAuth client secret
```

Không:

```dockerfile
ENV DATABASE_PASSWORD=...
```

Không:

```dockerfile
COPY .env .
```

Không hard-code credential trong source/Jenkinsfile.

Secret thuộc về backend, CI/CD hoặc secret store và chỉ được inject vào nơi cần sử dụng.

Nguyên tắc:

```text
secret exists
     ↓
only where needed
     ↓
only for as long as needed
     ↓
minimum permission needed
```

Đây là `least privilege` áp dụng cho credential scope và credential lifetime.

---

## 7. Scan bundle/image để tìm credential

Dùng fake marker thay vì credential thật:

```text
SESSION7_SUPER_SECRET_123456
```

Scan static files:

```bash
docker run --rm portal-frontend:$SHORT_SHA sh -c '
grep -R -n -E \
"BEGIN PRIVATE KEY|AWS_SECRET_ACCESS_KEY|DATABASE_PASSWORD|SESSION7_SUPER_SECRET_123456" \
/usr/share/nginx/html || true
'
```

Kiểm tra environment trong image:

```bash
docker image inspect \
  portal-frontend:$SHORT_SHA \
  --format '{{json .Config.Env}}'
```

Checkpoint: không có credential/fake secret marker trong bundle hoặc image config.

---

## 8. Cache policy cho runtime config

`runtime-config.json` có lifecycle khác hashed JS/CSS asset.

Nginx rule:

```nginx
location = /runtime-config.json {
    add_header Cache-Control "no-store";
    try_files $uri =404;
}
```

Kiểm tra:

```bash
curl -I http://localhost:8083/runtime-config.json
```

Kỳ vọng:

```text
Cache-Control: no-store
```

Failure mode cần nhớ:

```text
new image
+
stale runtime config
       ↓
frontend gọi sai API/environment
```

Nếu browser vẫn gọi API cũ sau khi deploy config mới, kiểm tra:

```text
1. response /runtime-config.json thực tế
2. Cache-Control / CDN / proxy / browser cache
3. frontend đang đọc đúng runtime config chưa
```

---

## 9. Runtime config phải được validate

TypeScript type không validate JSON từ network ở runtime.

```text
runtime-config.json
=
untrusted runtime input
```

App nên fail-fast khi config thiếu hoặc sai.

Ví dụ:

```ts
function assertNonEmpty(
  value: unknown,
  name: string,
): asserts value is string {
  if (typeof value !== "string" || value.trim() === "") {
    throw new Error(`Invalid runtime config: ${name}`);
  }
}
```

Config có `schemaVersion`:

```json
{
  "schemaVersion": 1
}
```

Mục đích:

```text
frontend v2 expects config schema v2
+
infra accidentally provides config v1
        ↓
FAIL FAST
```

thay vì app chạy với `undefined` và lỗi muộn.

---

## 10. Config/image compatibility

Một release production không chỉ gồm image.

Nên truy vết tối thiểu:

```text
Git commit
image digest
runtime config schema/version
public config version
```

Ví dụ release manifest tương lai:

```json
{
  "release": "2026.09.17.1",
  "gitCommit": "83a5598...",
  "image": "portal-frontend@sha256:...",
  "runtimeConfig": {
    "schemaVersion": 1,
    "environment": "production"
  }
}
```

Bài học:

```text
rollback application
!=
chỉ đổi Docker image
```

Có trường hợp phải rollback đúng cặp:

```text
artifact/image
+
runtime public config
```

Phần này sẽ nối sang Session 10 — Artifact & Rollback.

---

## 11. Build-time config vẫn có use case hợp lý

Không phải mọi build-time value đều xấu.

Phù hợp cho metadata xác định artifact:

```text
Git SHA
application version
build metadata
Sentry release ID
```

Ví dụ từ Session 6:

```dockerfile
ARG GIT_COMMIT
LABEL org.opencontainers.image.revision="${GIT_COMMIT}"
```

Khác với environment-specific runtime config như API URL.

---

## Incident questions — kết quả

### Backend đổi API URL

Không cần rebuild frontend image. Chỉ cần thay runtime public config nếu contract frontend không đổi.

### Runtime config thay nhưng checksum assets đổi

Không đúng với kiến trúc build-once/promote-many của lab; public runtime config không nên làm thay đổi JS bundle.

### `DATABASE_PASSWORD` nằm trong runtime-config.json

Không an toàn. Browser có thể tải/read file trực tiếp qua DevTools/network; HTTPS chỉ bảo vệ dữ liệu khi truyền trên mạng, không giấu dữ liệu khỏi chính browser/user nhận response.

### Image mới nhưng browser gọi API cũ

Kiểm tra runtime-config response và cache layer trước: browser/proxy/CDN có thể đang dùng stale config.

### Phân loại

```text
BUILD
- GIT_COMMIT

PUBLIC_RUNTIME
- API_BASE_URL
- ENTRA_CLIENT_ID

SECRET
- DATABASE_PASSWORD
- AWS_DEPLOY_TOKEN
```

---

## Checkpoint Session 7

Đã đạt:

- Hiểu build-time config và cách Vite nhúng `VITE_*` vào bundle.
- Chứng minh DEV/PROD build-time config tạo artifact khác nhau.
- Chuyển Entra/public frontend config sang `runtime-config.json`.
- Load runtime config trước khi initialize MSAL/React.
- Dùng cùng Docker image với hai environment khác nhau.
- Chứng minh bundle checksum giống nhau giữa DEV/PROD.
- Phân biệt public config và secret.
- Chứng minh `.env` không tự động là secret.
- Scan bundle/image để tìm fake credential marker.
- Thêm cache policy cho runtime config.
- Hiểu fail-fast validation và config schema version.
- Hiểu image/config compatibility và ảnh hưởng tới rollback.
- Phân loại đúng build metadata, public runtime config và secret.

## Next — Session 8: Docker Compose & Request Path

Tiếp theo:

```text
Browser
   ↓
Host port
   ↓
Nginx / frontend container
   ↓
Docker network
   ↓
API service
```

Mục tiêu Session 8:

- Dựng frontend + backend bằng một lệnh Compose.
- Hiểu service DNS và container-to-container networking.
- Phân biệt host port với container port.
- Browser gọi `/api` qua Nginx thay vì gọi Docker service hostname trực tiếp.
- Quan sát health check, logs, restart và SIGTERM.
- Dừng API để quan sát lỗi rồi khởi động lại để xác nhận phục hồi.
