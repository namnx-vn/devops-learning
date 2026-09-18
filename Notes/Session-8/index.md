# Session 8 — Docker Compose & Request Path

## Mục tiêu

Dựng frontend + backend bằng Docker Compose và hiểu chính xác đường đi của request:

```text
Browser / curl
   ↓
Docker host :8080
   ↓
frontend container / Nginx :80
   ↓
Docker internal network
   ↓
service DNS: api
   ↓
api container / Node :4000
```

Checkpoint của buổi:

```text
docker compose up
      ↓
frontend + API chạy
      ↓
FE/Nginx gọi API qua service DNS
      ↓
stop API → quan sát lỗi
      ↓
start API → phục hồi
      ↓
SIGTERM graceful shutdown
      ↓
restart policy self-recovery
```

---

## 1. Host port vs container port

Compose frontend:

```yaml
ports:
  - "8080:80"
```

```text
8080 = host port trên Ubuntu VM
80   = container port nơi Nginx listen
```

API không cần publish port ra host nếu chỉ Nginx gọi nó:

```yaml
api:
  expose:
    - "4000"
```

Do đó:

```text
VM → localhost:4000        ❌
frontend → api:4000        ✅
```

Phân biệt:

```text
Dockerfile EXPOSE 4000
→ metadata của image; không publish ra host

Compose expose: 4000
→ mô tả internal service port; không publish ra host

Compose ports: 4000:4000
→ publish HOST:CONTAINER
```

---

## 2. Docker Compose network và service DNS

Compose tạo default network:

```text
portal-saas_default
```

Các service cùng network có thể gọi nhau bằng service name:

```text
frontend → api:4000
```

Docker internal DNS resolve service name `api` sang IP hiện tại của API container. Không hard-code container IP vì IP có thể thay đổi khi recreate.

Browser ở ngoài Docker network không biết hostname `api`.

```text
Browser → http://api:4000     ❌

Browser → /api
        ↓
Nginx
        ↓
api:4000                     ✅
```

---

## 3. Compose architecture

```yaml
services:
  frontend:
    build:
      context: .
      dockerfile: apps/frontend/Dockerfile
    ports:
      - "8080:80"
    depends_on:
      api:
        condition: service_healthy

  api:
    build:
      context: .
      dockerfile: apps/backend/Dockerfile

    restart: unless-stopped

    expose:
      - "4000"

    environment:
      NODE_ENV: test
      RESEND_API_KEY: session8-fake-resend-key

    healthcheck:
      test:
        [
          "CMD",
          "node",
          "-e",
          "fetch('http://localhost:4000/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"
        ]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 5s
```

Fake marker được dùng cho lab; không commit credential thật vào Compose.

---

## 4. Backend Dockerfile

```dockerfile
FROM node:24-alpine

RUN npm install -g pnpm@10.34.5

WORKDIR /app

COPY package.json ./
COPY pnpm-lock.yaml ./
COPY pnpm-workspace.yaml ./
COPY apps/backend/package.json ./apps/backend/package.json

RUN pnpm install --filter backend --frozen-lockfile

COPY apps/backend ./apps/backend

WORKDIR /app/apps/backend

RUN pnpm exec prisma generate

EXPOSE 4000

CMD ["pnpm", "exec", "tsx", "src/index.ts"]
```

Không dùng `tsx watch` làm runtime command vì container runtime không cần source watcher.

---

## 5. Nginx reverse proxy /api

Frontend runtime config dùng:

```json
{
  "apiBaseUrl": "/api"
}
```

Nginx:

```nginx
location /api/ {
    proxy_pass http://api:4000/;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

Browser gọi `/api/health`; do trailing slash trong `proxy_pass http://api:4000/;`, backend nhận `/health`.

```text
curl http://localhost:8080/api/health
             ↓
Ubuntu :8080
             ↓
frontend :80
             ↓
Nginx location /api/
             ↓
Docker DNS "api"
             ↓
api:4000
             ↓
Express GET /health
             ↓
200 {"ok":true}
```

---

## 6. Healthcheck và depends_on

```text
container started
!=
application ready
```

Có thể container `Up` nhưng application `unhealthy`.

Lab dùng:

```yaml
depends_on:
  api:
    condition: service_healthy
```

Flow:

```text
start api
   ↓
healthcheck /health
   ↓
healthy
   ↓
start frontend
```

Healthcheck chạy bên trong API container nên `http://localhost:4000/health` là đúng tại đó.

---

# Incidents thực hành

## Incident 1 — API exit vì missing runtime environment

Symptom:

```text
container portal-saas-api-1 exited (1)
dependency api failed to start
```

Log:

```text
Error: Missing API key.
at new Resend(...)
at apps/backend/src/lib/mailer.ts
```

Root cause: `mailer.ts` tạo Resend client ngay lúc import nhưng container chưa có `RESEND_API_KEY`.

```text
container starts
 ↓
Node imports modules
 ↓
mailer.ts
 ↓
missing runtime env
 ↓
Node exit 1
 ↓
container Exited
 ↓
healthcheck không thể healthy
 ↓
frontend bị depends_on chặn
```

Fix lab: inject fake runtime value và `NODE_ENV=test`. TypeScript non-null assertion `!` không tạo giá trị runtime.

---

## Incident 2 — host port 8080 already allocated

Symptom:

```text
Bind for 0.0.0.0:8080 failed: port is already allocated
```

Check:

```bash
sudo ss -ltnp | grep :8080
docker ps --format 'table {{.Names}}\t{{.Ports}}' | grep 8080
```

Evidence cho thấy container cũ `portal-frontend` đang giữ `0.0.0.0:8080->80/tcp`.

Fix:

```bash
docker stop portal-frontend
```

```text
port conflict
!=
application error
!=
Docker DNS error
```

Layer lỗi là host port publishing.

---

## Incident 3 — Nginx cannot resolve service name api

Symptom:

```text
host not found in upstream "api"
```

Inspect API:

```bash
docker inspect portal-saas-api-1 \
  --format '{{json .NetworkSettings.Networks}}'
```

API attach `portal-saas_default` và có alias `api`.

Inspect frontend trả:

```json
{}
```

=> frontend container cũ không attach network.

```text
api      → portal-saas_default ✅
frontend → no network          ❌
```

Clean recovery:

```bash
docker compose down
docker compose up -d
```

Sau recreate, cả hai service cùng network và request path hoạt động.

---

## 7. 502 vs connection refused

Khi Nginx đang chạy nhưng API bị stop:

```bash
docker compose stop api
curl -i http://localhost:8080/api/health
```

```text
curl → Nginx ✅
Nginx → API ❌
→ 502 Bad Gateway
```

Nếu frontend/Nginx chết:

```text
curl localhost:8080
→ curl (7) Failed to connect
```

```text
curl (7)
→ chưa tới Nginx / không có host listener

502
→ đã tới reverse proxy
→ upstream có vấn đề
```

---

## 8. Recovery khi API quay lại

```bash
docker compose start api
```

Đợi API healthy rồi `curl /api/health` trở lại HTTP 200. Không cần restart frontend trong stop/start case này.

---

## 9. SIGTERM và graceful shutdown

```text
docker compose stop api
      ↓
SIGTERM
      ↓
application cleanup
      ↓
nếu không thoát trong timeout
      ↓
SIGKILL
```

Backend thêm graceful shutdown:

```ts
const server = app.listen(PORT, () => {
  console.log(`Backend running on port ${PORT}`);
});

process.on("SIGTERM", () => {
  console.log("SIGTERM received, shutting down...");

  server.close(async () => {
    console.log("HTTP server closed");
    await prisma.$disconnect();
    console.log("Database disconnected");
    process.exit(0);
  });
});
```

Log lab:

```text
Backend running on port 4000
SIGTERM received, shutting down...
HTTP server closed
Database disconnected
api-1 exited with code 0
```

`SIGKILL` không cho application cơ hội cleanup.

---

## 10. Image content vs runtime config

```text
sửa source code
→ image content đổi
→ phải docker compose build api

sửa environment / restart policy
→ runtime configuration đổi
→ image không đổi
```

---

## 11. Healthcheck vs restart policy

```text
healthcheck
→ application còn hoạt động đúng không?

restart policy
→ container process chết thì Docker làm gì?
```

Nếu Node còn sống nhưng `/health` fail: container có thể `unhealthy`; healthcheck tự nó không restart container.

Nếu main process chết và dùng:

```yaml
restart: unless-stopped
```

Docker restart container. Manual `docker compose stop api` được coi là operator intent.

---

## 12. Crash test và PID namespace nuance

Test `docker compose exec api sh -c 'kill -9 1'` không tạo crash như kỳ vọng trong lab; `RestartCount` vẫn 0.

Test từ Ubuntu host:

```bash
HOST_PID=$(docker inspect portal-saas-api-1 --format '{{.State.Pid}}')
sudo kill -9 "$HOST_PID"
```

Với `restart: unless-stopped`, Docker khởi động lại container.

Verify:

```bash
docker inspect portal-saas-api-1 \
  --format 'Status={{.State.Status}} RestartCount={{.RestartCount}}'
```

Sau recovery, `/api/health` trở lại HTTP 200.

---

# Command reference — Session 8

```bash
# validate
docker compose config

# build
docker compose build
docker compose build api

# lifecycle
docker compose up -d
docker compose up -d api
docker compose start api
docker compose stop api
docker compose down

# status
docker compose ps
docker compose ps -a

# logs
docker compose logs api
docker compose logs --tail=100 frontend
docker compose logs -f api

# networking
docker inspect <container> --format '{{json .NetworkSettings.Networks}}'
docker network inspect portal-saas_default

# host listener / port owner
sudo ss -ltnp | grep :8080
docker ps --format 'table {{.Names}}\t{{.Ports}}' | grep 8080

# request path
curl -i http://localhost:8080/api/health

# restart policy
docker inspect portal-saas-api-1 \
  --format 'RestartPolicy={{.HostConfig.RestartPolicy.Name}} RestartCount={{.RestartCount}}'

# host PID
docker inspect portal-saas-api-1 --format '{{.State.Pid}}'
```

---

# Checkpoint Session 8

Đã đạt:

- Dựng frontend + backend bằng Docker Compose.
- Hiểu `HOST:CONTAINER` port mapping.
- Phân biệt Dockerfile `EXPOSE`, Compose `expose`, và `ports`.
- API không publish trực tiếp ra host.
- Hiểu default Compose network và service DNS `api:4000`.
- Browser gọi `/api` qua Nginx.
- Hiểu trailing slash của `proxy_pass`.
- Dùng healthcheck và `depends_on: service_healthy`.
- Phân biệt container Up và application healthy.
- Debug missing runtime environment.
- Debug host port conflict bằng `ss` và `docker ps`.
- Debug Docker network/service DNS bằng `docker inspect`.
- Phân biệt connection refused và HTTP 502.
- Stop API để quan sát failure path; start lại để xác nhận recovery.
- Thực hành SIGTERM graceful shutdown.
- Phân biệt image content và runtime configuration.
- Hiểu healthcheck khác restart policy.
- Dùng `restart: unless-stopped` và thực hành crash/recovery.

## Next — Session 9: Cache & Release Frontend

```text
index.html / runtime-config.json
→ revalidate / no-store

hashed JS/CSS assets
→ long cache + immutable
```

Checkpoint Session 9:

- Xây cache matrix cho HTML, runtime config và hashed assets.
- Xác nhận header bằng `curl` và browser DevTools.
- Phân biệt browser cache, proxy/CDN cache và service worker.
- Đảm bảo missing JS asset không bị SPA fallback trả `index.html`.
