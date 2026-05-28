---
phase_id: PHASE-001
title: 基础层：代理 + Auth
status: draft
priority: P0
---

## Goal

建立 Next.js → Python 后端的代理通道，并实现 Auth 统一（better-auth 会话自动映射为 Python JWT），使后续所有大A功能模块可以调用 Python 后端而无需用户二次登录。

## In Scope

- REQ-001：项目脚手架（docs/tasks/harness 目录结构，`package.json` 补充 typecheck 脚本）
- REQ-002：Next.js API 代理层（`app/api/cn/[...path]/route.ts`）
- REQ-003：Auth 统一（服务端 `CN_PASSPHRASE` → Python JWT，缓存于服务端 cookie）

## Out of Scope

- 任何大A功能页面（PHASE-002 起）
- TanStack Query 集成（PHASE-002）
- Docker 配置（PHASE-005）

## Exit Criteria

1. `curl http://localhost:3000/api/cn/health` 返回 `{"status": "ok"}`（需 Python 后端同时运行）
2. 未登录用户访问 `/api/cn/*` 收到 401，登录用户收到 Python 后端的真实响应
3. `pnpm typecheck && pnpm test` 通过；`pnpm lint` 对本 PR 改动文件无新增错误（全量 lint 基线清理见 REQ-001）

## Dependencies

- Python 后端（my-invest-global）在本地 port 8000 运行
- `.env` 中配置 `PYTHON_BACKEND_URL` 和 `CN_PASSPHRASE`

## Notes

代理层采用 Next.js Route Handler（App Router），不使用 `next.config.ts` 的 `rewrites`，原因：
- rewrites 不支持动态修改请求头（无法附加 JWT）
- Route Handler 可以在服务端处理 JWT 刷新逻辑，客户端不感知
