# Session 6 — Docker Image có thể dựng lại

## Mục tiêu

Đóng gói SPA thành Docker image có thể truy vết và dựng lại từ clean checkout.

Mental model:

```text
Git commit
   +
lockfile
   +
build tools
   +
base image
   +
Dockerfile
   ↓
Build stage (Node/pnpm)
   ↓
dist/
   ↓
Runtime stage (Nginx)
   ↓
versioned Docker image
```

---

## 1. Project và build context

Project thực hành: `portal-saas`, pnpm monorepo:

```text
portal-saas/
├── package.json
├── pnpm-lock.yaml
├── pnpm-workspace.yaml
└── apps/
    ├── frontend/   # React + Vite
    └── backend/
```

Dockerfile frontend nằm tại:

```text
apps/frontend/Dockerfile
```

nhưng build từ repo root:

```bash
docker build \
  -f apps/frontend/Dockerfile \
  -t portal-frontend:debug \
  .
```

Điểm cần nhớ:

```text
Dockerfile location != build context
```

`.` là build context. `COPY` chỉ đọc được file nằm trong context.

---

## 2. Multi-stage build

Build stage cần Node/pnpm/source để tạo `dist`:

```dockerfile
FROM node:24-alpine@sha256:<pinned-digest> AS build

RUN npm install -g pnpm@10.34.5
WORKDIR /app

COPY package.json ./
COPY pnpm-lock.yaml ./
COPY pnpm-workspace.yaml ./
COPY apps/frontend/package.json ./apps/frontend/package.json

RUN pnpm install --filter frontend --frozen-lockfile

COPY apps/frontend ./apps/frontend
RUN pnpm --filter frontend build
```

Runtime stage chỉ cần Nginx + static artifact:

```dockerfile
FROM nginx:alpine@sha256:<pinned-digest> AS runtime

ARG GIT_COMMIT

LABEL org.opencontainers.image.revision="${GIT_COMMIT}"
LABEL org.opencontainers.image.source="https://github.com/namnx-vn/portal-saas"

COPY apps/frontend/nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/apps/frontend/dist /usr/share/nginx/html

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Mental model:

```text
Build stage
├── Node
├── pnpm
├── source
└── dist

        dist only
           ↓

Runtime stage
├── Nginx
└── static files
```

Node không cần tồn tại trong runtime image của SPA tĩnh.

---

## 3. Lockfile và layer cache

Repo dùng:

```text
pnpm-lock.yaml
pnpm@10.34.5
```

Cài dependency:

```bash
pnpm install --filter frontend --frozen-lockfile
```

Dockerfile copy package metadata trước source:

```text
COPY package/lock metadata
        ↓
RUN pnpm install
        ↓
COPY frontend source
        ↓
RUN build
```

Khi chỉ đổi source:

```text
pnpm install layer → CACHED
COPY source         → rebuild
Vite build          → rebuild
```

=> không cài lại dependency nếu package metadata/lockfile không đổi.

---

## 4. Incident thực tế: filename case-sensitive

Build Docker/Linux fail:

```text
Could not resolve "../assets/svg/sso_icon.svg"
```

Code import:

```text
sso_icon.svg
```

file trong Git:

```text
sso_Icon.svg
```

Linux phân biệt hoa/thường:

```text
sso_icon.svg != sso_Icon.svg
```

macOS có thể không lộ lỗi nếu filesystem đang case-insensitive.

Bài học:

> Clean Linux build giúp phát hiện assumption ẩn từ máy developer.

---

## 5. Tag, Image ID và digest

Tag là tên/pointer:

```text
portal-frontend:debug
portal-frontend:<git-sha>
```

Một image có thể có nhiều tag.

Không nên dùng `latest` làm thông tin rollback duy nhất vì tag có thể đổi target.

Base image tag:

```dockerfile
FROM node:24-alpine
```

là mutable: registry có thể cập nhật tag để trỏ sang digest mới.

Pin digest:

```dockerfile
FROM node:24-alpine@sha256:<digest>
```

=> cố định exact base image content.

Trade-off:

```text
Pin digest
+ reproducible hơn
+ dễ audit/trace
- không tự nhận security update
→ cần quy trình/bot update digest có kiểm soát
```

---

## 6. Gắn Git SHA vào image

Lấy revision:

```bash
FULL_SHA=$(git rev-parse HEAD)
SHORT_SHA=$(git rev-parse --short=12 HEAD)
```

Build:

```bash
docker build \
  --build-arg GIT_COMMIT="$FULL_SHA" \
  -f apps/frontend/Dockerfile \
  -t portal-frontend:$SHORT_SHA \
  .
```

Inspect provenance:

```bash
docker image inspect portal-frontend:$SHORT_SHA \
  --format 'ID={{.Id}} REVISION={{ index .Config.Labels "org.opencontainers.image.revision" }}'
```

Checkpoint đạt:

```text
Git revision:
83a5598d455affe4ea30bb3c32599f1e99892d9c

Image metadata revision:
83a5598d455affe4ea30bb3c32599f1e99892d9c
```

=> image truy ngược được đúng source commit.

---

## 7. Clean checkout và reproducible artifact

Build lại từ fresh clone, không dùng Docker cache:

```bash
docker build \
  --no-cache \
  --build-arg GIT_COMMIT="$FULL_SHA" \
  -f apps/frontend/Dockerfile \
  -t portal-frontend:${SHORT_SHA}-clean \
  .
```

Checksum static artifact:

```bash
docker run --rm portal-frontend:${SHORT_SHA}-clean sh -c '
  cd /usr/share/nginx/html &&
  find . -type f -exec sha256sum {} \; |
  sort |
  sha256sum
'
```

Kết quả clean build:

```text
0a52dc7481199514ee0e2bc4eff2553fdf841dc1e71a0b0cd8380e398da2a82b
```

Checksum này giống build trước đó.

Bằng chứng:

```text
fresh checkout
+
fresh dependency install
+
--no-cache
+
same controlled inputs
↓
same frontend artifact checksum
```

Image ID không bắt buộc giống artifact checksum; image còn chứa metadata, filesystem và Docker config. Với SPA, checksum `/usr/share/nginx/html` giúp kiểm tra artifact thực sự được serve.

---

## 8. `.dockerignore`

Nên có tại repo root vì build context là `.`:

```dockerignore
.git
.gitignore
.DS_Store

**/node_modules
**/dist
**/.env
**/.env.*

cookies.txt
npm-debug.log*
pnpm-debug.log*
```

Mục đích:

```text
smaller build context
+
không gửi file không cần thiết / .env vào Docker build context
```

Follow-up sau session: bảo đảm `.dockerignore` được commit vào `portal-saas`; fresh clone cuối buổi cho thấy file này chưa hiện diện đúng như bản lab ban đầu.

---

## 9. Reproducible build — cách debug khi checksum khác

Nếu cùng commit nhưng:

```text
Build A → checksum AAA
Build B → checksum BBB
```

sau khi base image và lockfile đã pin, kiểm tra các input không deterministic:

```text
ENV khác nhau
random/generated content
build timestamp
external API/resource
build tool chưa pin
file ngoài Git lọt vào context
```

Nguyên tắc:

> Same Git commit chưa đủ để đảm bảo same artifact nếu còn input build bên ngoài chưa được kiểm soát.

---

## Checkpoint Session 6

Đã đạt:

- Multi-stage Dockerfile: build Node/pnpm → runtime Nginx.
- Hiểu build context và `COPY`.
- Dùng pnpm lockfile + `--frozen-lockfile`.
- Quan sát Docker layer cache thực tế.
- Phát hiện lỗi case-sensitive khi build Linux.
- Phân biệt mutable tag và pinned digest.
- Pin base image bằng digest.
- Tag image theo Git SHA.
- Ghi full Git revision vào OCI label.
- Build từ fresh checkout với `--no-cache`.
- Xác nhận artifact checksum ổn định.

Follow-up nhỏ:

- Commit `.dockerignore` vào repo `portal-saas`.

## Next — Session 7: Config & Secrets

Tiếp theo:

```text
build-time config
vs
runtime public config
vs
secret
```

Mục tiêu Session 7:

```text
same frontend artifact checksum
        +
runtime-config A / B
        ↓
chạy được nhiều environment
```

và chứng minh bundle/image frontend không chứa credential.