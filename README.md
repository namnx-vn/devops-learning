# DevOps Learning

Lộ trình DevOps cá nhân hóa cho Senior Frontend / Tech Lead.

## Current progress

```text
Completed: 7 / 30 sessions
Progress : 23%
Week 1   : ✅ Sessions 1–5
Week 2   : ✅ Sessions 6–7
Next     : Session 8 — Compose & Request Path
```

Session 7 đã hoàn thành checkpoint **Config & Secrets**:

- Phân biệt build-time metadata, runtime public config và secret.
- Chứng minh `VITE_*` được nhúng vào frontend bundle khi build.
- Chuyển public frontend config sang `runtime-config.json`.
- Cùng Docker image chạy DEV/PROD bằng hai runtime config khác nhau.
- Bundle checksum giữ nguyên giữa hai environment.
- Runtime config được validate/fail-fast và có `schemaVersion`.
- `runtime-config.json` có cache policy riêng để tránh stale config.
- Bundle/image được scan và không chứa fake credential marker.
- Hiểu image/config compatibility và tác động tới rollback.

## Learning material

- [6-week learning plan](./devops-learning-plan.html)
- [Week 1 master summary](./Notes/index.md)
- [Session 6 — Docker Image có thể dựng lại](./Notes/Session-6/index.md)
- [Session 7 — Config & Secrets](./Notes/Session-7/index.md)
- [Debug command reference](./Notes/debug-command-reference.md)

## Next session

**Session 8 — Compose & Request Path**

Mục tiêu:

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

Dựng frontend + backend bằng một lệnh Compose, hiểu service DNS/port mapping/request path, và thực hành dừng API để quan sát lỗi rồi phục hồi.
