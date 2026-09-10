# Session 1 — Web Flow, Jenkins Basics & Ubuntu Environment

## Mục tiêu buổi học

- Chọn một ứng dụng cá nhân để dùng xuyên suốt khóa học DevOps.
- Chuẩn bị Ubuntu VM làm môi trường lab.
- Hiểu luồng request cơ bản của một ứng dụng web.
- Đọc được cấu trúc cơ bản của một `Jenkinsfile`.
- Bắt đầu nối các khái niệm Networking, Nginx, Docker và application port.

---

## 1. Ubuntu Environment

Đã hoàn thành phần chuẩn bị Ubuntu bằng Multipass và chạy thử một số Linux command cơ bản.

Ví dụ:

```bash
whoami
uname -a
ip a
```

Cập nhật package:

```bash
sudo apt update && sudo apt upgrade -y
```

### Kết quả

- Ubuntu VM hoạt động.
- Có thể truy cập VM từ Mac Terminal.
- Đã bắt đầu làm quen với Linux command và networking trên Ubuntu.

---

## 2. Jenkins Pipeline — Các thành phần cơ bản

Một pipeline đơn giản có thể có flow:

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
Manual Approval
    ↓
Deploy
```

### `agent`

`agent` là nơi Jenkins thực thi các task/command của pipeline.

Ví dụ:

```groovy
pipeline {
    agent any
}
```

Có thể hiểu:

```text
Jenkins Controller
       ↓ giao job
Jenkins Agent
       ↓
npm ci / npm test / npm run build / docker build ...
```

> Jenkins Agent có thể là VM, physical server hoặc container.

---

### `checkout scm`

```groovy
stage('Checkout') {
    steps {
        checkout scm
    }
}
```

Dùng để lấy source code từ SCM như GitHub/GitLab về workspace của Jenkins Agent.

```text
GitHub / GitLab
      ↓
Jenkins Agent Workspace
```

---

### `npm ci`

```groovy
stage('Install Dependencies') {
    steps {
        sh 'npm ci'
    }
}
```

Command được chạy trên Jenkins Agent.

Trong CI thường ưu tiên:

```bash
npm ci
```

thay vì:

```bash
npm install
```

vì `npm ci` cài dependency dựa theo `package-lock.json`, giúp build reproducible hơn.

---

### Test và Exit Code

```groovy
stage('Test') {
    steps {
        sh 'npm test'
    }
}
```

Linux convention:

```text
exit code 0     → success
exit code != 0  → failure
```

Ví dụ:

```text
npm test
   ↓
exit 1
   ↓
Test stage FAILED
   ↓
Pipeline dừng / các stage sau bị skip theo flow mặc định
```

---

### Build & Artifact

Với React/Vite:

```bash
npm run build
```

thường tạo:

```text
dist/
```

`dist/` là build artifact có thể được lưu lại hoặc dùng để deploy.

```text
Source Code
    ↓
Build
    ↓
dist/
    ↓
Artifact
```

---

## 3. Credentials & Secrets

Không hard-code secret trực tiếp vào source code hoặc `Jenkinsfile`.

Sai:

```groovy
DEPLOY_TOKEN = "my-secret-token"
```

Thay vào đó, secret nên được lưu trong hệ thống quản lý credentials/secrets và inject khi runtime cần dùng.

Ví dụ:

```text
AWS Secrets Manager
        ↓
CI/CD / Application Runtime
        ↓
Environment Variable
```

Hoặc:

```text
Jenkins Credentials Store
        ↓
Pipeline
        ↓
Environment Variable
```

### Nguyên tắc

```text
Secret không commit vào Git.
Secret chỉ được inject khi cần.
```

---

## 4. Manual Approval

Trước khi deploy production có thể thêm một manual gate:

```groovy
stage('Approval') {
    steps {
        input message: 'Deploy to production?',
              ok: 'Deploy'
    }
}
```

Flow:

```text
Test ✅
Build ✅
Artifact ✅
    ↓
Human Approval
    ↓
Production
```

Approval không đảm bảo production chắc chắn không lỗi. Nó chỉ thêm một control point trước một hành động có rủi ro cao.

Về sau cần kết hợp thêm:

- Monitoring
- Health Check
- Rollback
- Blue/Green Deployment
- Canary Deployment

---

## 5. Luồng khi Browser gọi một Website

Ví dụ:

```text
https://app.example.com
```

Flow cơ bản:

```text
Browser
   ↓
DNS
   ↓
IP Address
   ↓
TCP Connection
   ↓
TLS Handshake
   ↓
HTTP Request
   ↓
Web Server / Application
   ↓
HTTP Response
   ↓
Browser Render
```

Có thể ghi nhớ ngắn gọn:

```text
DNS → Connection/TLS → HTTP → Application
```

### Vai trò

```text
DNS  → Domain → IP
TLS  → tạo kết nối mã hóa và xác thực server
HTTP → Request / Response
App  → xử lý request và trả dữ liệu
```

---

## 6. Port 443 và HTTPS

Một điểm cần nhớ:

```text
443 ≠ bắt buộc cho HTTPS
443 = default port của HTTPS
```

Về kỹ thuật có thể chạy:

```text
https://example.com:8443
```

nếu service tại port đó hỗ trợ TLS đúng cách.

Trong production thường dùng:

```text
Browser
   ↓ HTTPS :443
Nginx
   ↓
Application :3000
```

---

## 7. Nginx làm Reverse Proxy

Một pattern phổ biến:

```text
Internet
   ↓
Nginx :443
   ↓
Node.js :3000
```

Nginx có thể đảm nhiệm:

- Reverse proxy
- TLS termination
- Routing
- Load balancing
- Serve static files
- Compression
- Rate limiting
- Security headers

Ví dụ:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:3000;
}
```

Node.js không nhất thiết phải expose port `3000` ra Internet nếu chỉ Nginx cần truy cập nó.

```text
Internet → :3000 ❌
Internet → :443  ✅
Nginx    → :3000 ✅
```

Lợi ích là giảm attack surface và tránh client bypass reverse proxy.

---

## 8. `127.0.0.1` vs `0.0.0.0` trong Container

### `127.0.0.1`

Nếu Node.js trong container bind:

```text
127.0.0.1:3000
```

thì chỉ process trong chính container đó truy cập được.

Một container Nginx khác sẽ không truy cập được port này qua Docker network.

### `0.0.0.0`

Nếu app bind:

```js
app.listen(3000, '0.0.0.0')
```

thì app listen trên các network interface của container và container khác trong cùng Docker network có thể kết nối.

```text
Nginx container
       ↓
Docker Network
       ↓
API container :3000
```

### Ghi nhớ

```text
127.0.0.1 = chỉ loopback của chính container
0.0.0.0   = listen trên mọi interface của container
```

`0.0.0.0:3000` không đồng nghĩa port `3000` đã public ra Internet.

---

## 9. Docker Port Mapping

Ví dụ:

```yaml
ports:
  - "8080:3000"
```

Cú pháp:

```text
HOST_PORT : CONTAINER_PORT
```

Do đó:

```text
8080 = port trên host
3000 = port trong container
```

Flow:

```text
Client
   ↓
Host :8080
   ↓ Docker port mapping
Container :3000
   ↓
Node.js
```

Ghi nhớ:

```text
8080:3000
^^^^ ^^^^
host container
```

---

## 10. Docker Internal Networking

Nếu `nginx` và `api` nằm trong cùng Docker Compose network thì Nginx có thể gọi:

```text
http://api:3000
```

mà `api` không cần publish port `3000` ra host.

Ví dụ:

```yaml
services:
  nginx:
    image: nginx
    ports:
      - "443:443"

  api:
    build: .
```

Node.js:

```js
app.listen(3000, '0.0.0.0')
```

Docker Compose cung cấp DNS nội bộ cho các service:

```text
api
 ↓ Docker DNS
IP nội bộ của API container
```

Flow production phổ biến:

```text
Internet
   ↓
Host :443
   ↓
Nginx Container
   ↓
Docker Internal Network
   ↓
API Container :3000
```

Port `3000` không cần public ra host.

---

## 11. Bức tranh tổng thể đã học

Các kiến thức bắt đầu kết nối với nhau:

```text
Developer
   ↓ git push
GitHub
   ↓
Jenkins
   ↓
Test / Build
   ↓
Artifact
   ↓
Approval
   ↓
Deploy
```

Khi user truy cập ứng dụng:

```text
Browser
   ↓
DNS
   ↓
Server IP
   ↓
HTTPS :443
   ↓
Nginx
   ↓
Docker Network
   ↓
Application :3000
```

Đây là nền tảng để tiếp tục học:

```text
Git
→ Jenkins / CI/CD
→ Docker
→ Linux
→ Networking
→ Nginx
→ Deployment
```

---

## 12. Self Assessment — Session 1

### Tự làm / đã hiểu

- [x] Setup Ubuntu VM.
- [x] Truy cập Ubuntu từ Mac Terminal.
- [x] Hiểu `agent` là nơi Jenkins thực thi command.
- [x] Hiểu `checkout scm` lấy source code từ Git/SCM.
- [x] Hiểu `npm ci` chạy trên Jenkins Agent.
- [x] Hiểu test trả exit code khác `0` có thể làm pipeline fail.
- [x] Biết React/Vite thường build ra `dist/`.
- [x] Hiểu không hard-code secrets vào repository.
- [x] Có kinh nghiệm với AWS Secrets Manager.
- [x] Hiểu mục đích của manual approval trước production.
- [x] Hiểu flow `DNS → Connection/TLS → HTTP → Application`.
- [x] Hiểu Nginx có thể đứng trước application làm reverse proxy.
- [x] Hiểu Docker port mapping `HOST_PORT:CONTAINER_PORT`.
- [x] Hiểu container cùng network có thể giao tiếp bằng service name.

### Điểm đã sửa trong quá trình học

- [x] HTTPS không bắt buộc phải dùng port `443`; đây là default port.
- [x] Với `8080:3000`: `8080` là host port, `3000` là container port.
- [x] Container khác không truy cập được service chỉ bind vào `127.0.0.1` của container.
- [x] Service muốn nhận connection từ container khác thường cần listen trên `0.0.0.0`.

### Cần học tiếp

- [ ] Docker image vs container.
- [ ] Dockerfile.
- [ ] Docker Compose chi tiết hơn.
- [ ] Docker network thực tế.
- [ ] Nginx reverse proxy thực hành.
- [ ] Jenkins pipeline chạy thật.
- [ ] CI/CD deployment flow hoàn chỉnh.
- [ ] Networking: port, interface, firewall và routing.

---

## Kết luận Session 1

Buổi 1 đã hoàn thành phần môi trường Ubuntu và nắm được nền tảng của một pipeline CI/CD cũng như request flow của ứng dụng web.

Mental model quan trọng nhất sau buổi học:

```text
Code
 ↓
Git
 ↓
Jenkins Agent
 ↓
Test
 ↓
Build
 ↓
Artifact
 ↓
Deploy

User Request
 ↓
DNS
 ↓
HTTPS/TLS
 ↓
Nginx
 ↓
Docker Network
 ↓
Application
```

Session tiếp theo có thể bắt đầu đi sâu vào Docker và biến các khái niệm networking ở trên thành lab thực tế.
