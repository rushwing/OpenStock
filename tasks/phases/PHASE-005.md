---
phase_id: PHASE-005
title: Docker 容器化 & CI/CD
status: draft
priority: P2
---

## Goal

将整个平台打包为 Docker Compose 应用，支持本地一键启动和树莓派部署，并配置 GitHub Actions CI/CD 自动化。

## In Scope

- REQ-011：Docker 容器化（Next.js Dockerfile, Python Dockerfile, docker-compose.yml, .dockerignore）
- `next.config.ts` 添加 `output: 'standalone'`
- GitHub Actions workflow（build → test → push image → deploy）
- Nginx 反向代理配置示例（可选，用于域名访问）
- Cloudflare Tunnel 配置说明（可选，用于树莓派公网访问）

## Out of Scope

- 实际部署操作（文档提供步骤，操作由 human-001 完成）

## Exit Criteria

1. `docker compose up --build` 在本地 Mac 成功启动 Next.js + Python FastAPI
2. 访问 `http://localhost:3000` 正常渲染，`/api/cn/health` 返回 200
3. Next.js Docker 镜像大小 < 500MB（利用 standalone 输出）
4. Python Docker 镜像大小 < 1GB（多阶段构建，不含 dev 依赖）
5. `README.md` 包含本地启动和树莓派部署完整步骤

## Dependencies

- PHASE-004 完成（功能完整可测试）

## Notes

Python 镜像需处理 ARM64 兼容性（树莓派）。使用 `docker buildx build --platform linux/amd64,linux/arm64` 构建多架构镜像，推送到 GitHub Container Registry（GHCR）。

Neo4j 在树莓派上内存占用约 500MB，若内存紧张可通过环境变量 `ENABLE_NEO4J=false` 跳过启动（降级为无 KG 模式）。
