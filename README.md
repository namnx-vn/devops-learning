# DevOps Learning

Lộ trình DevOps cá nhân hóa cho Senior Frontend / Tech Lead.

## Current progress

```text
Completed: 9 / 30 sessions
Progress : 30%
Week 1   : ✅ Sessions 1–5
Week 2   : ✅ Sessions 6–9
Next     : Session 10 — Artifact & Rollback
```

Session 9 đã hoàn thành checkpoint **Cache & Release Frontend**:

- Xây cache matrix cho `index.html`, `runtime-config.json` và hashed assets.
- Hiểu `no-cache`, `no-store`, `max-age`, `immutable`.
- Thực hành ETag revalidation và HTTP `304 Not Modified`.
- Fix missing JS asset bị SPA fallback thành `index.html`; asset missing trả `404`.
- Hiểu safe release: **assets first, HTML last**.
- Hiểu lazy chunk failure và lý do cần old asset retention.
- Phân biệt cache retention với origin asset retention.
- Phân biệt browser cache, Service Worker, CDN/proxy và origin.
- Hiểu targeted CDN invalidation và cache-policy prevention.
- Xác định Docker frontend hiện tại chưa giữ old assets giữa các release.

## Learning material

- [6-week learning plan](./devops-learning-plan.html)
- [Week 1 master summary](./Notes/index.md)
- [Session 6 — Docker Image có thể dựng lại](./Notes/Session-6/index.md)
- [Session 7 — Config & Secrets](./Notes/Session-7/index.md)
- [Session 8 — Docker Compose & Request Path](./Notes/Session-8/index.md)
- [Session 9 — Cache & Release Frontend](./Notes/Session-9/index.md)
- [Debug command reference](./Notes/debug-command-reference.md)

## Next session

**Session 10 — Artifact & Rollback**

Tiếp tục từ cache/release lifecycle sang release identity và rollback:

```text
Git commit
   ↓
versioned artifact / image
   ↓
runtime config compatibility
   ↓
release manifest
   ↓
deploy
   ↓
verify
   ↓
rollback khi cần
```

Mục tiêu là biết chính xác một release gồm những artifact/config nào, giữ version cũ bao lâu và rollback an toàn thay vì chỉ đổi một tag hoặc rebuild lại source.
