# DevOps Learning

Lộ trình DevOps cá nhân hóa cho Senior Frontend / Tech Lead.

## Current progress

```text
Completed: 6 / 30 sessions
Progress : 20%
Week 1   : ✅ Sessions 1–5
Week 2   : ✅ Session 6
Next     : Session 7 — Config & Secrets
```

Session 6 đã hoàn thành checkpoint **reproducible frontend image**:

- Multi-stage Docker build: Node/pnpm build stage → Nginx runtime stage.
- Lockfile + `pnpm --frozen-lockfile`.
- Docker layer cache và build context.
- Base image pin bằng digest.
- Image tag + OCI label gắn Git commit SHA.
- Fresh checkout + `--no-cache` build.
- Artifact checksum ổn định: `0a52dc7481199514ee0e2bc4eff2553fdf841dc1e71a0b0cd8380e398da2a82b`.

Follow-up nhỏ trước/đầu Session 7: bảo đảm root `.dockerignore` được commit trong `portal-saas`.

## Learning material

- [6-week learning plan](./devops-learning-plan.html)
- [Week 1 master summary](./Notes/index.md)
- [Session 6 — Docker Image có thể dựng lại](./Notes/Session-6/index.md)
- [Debug command reference](./Notes/debug-command-reference.md)

## Next session

**Session 7 — Config & Secrets**

Mục tiêu:

```text
build-time config
vs
runtime public config
vs
secret
```

Dùng cùng frontend artifact với nhiều public runtime config và kiểm tra bundle/image không chứa credential.
