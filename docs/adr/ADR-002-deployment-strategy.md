# ADR-002：部署策略与成本分析

- **状态**：已采纳
- **日期**：2026-05-29
- **决策者**：Daniel Wong

---

## 背景

合并后的平台包含两个运行时进程：
1. **Next.js**（Node.js）— 前端 + API Routes + 代理层
2. **Python FastAPI**（uvicorn）— 大A数据处理引擎 + AI 信号

此外依赖以下存储服务：
- MongoDB Atlas（用户 Auth，已有云端实例）
- DuckDB（单文件，随 Python 进程部署）
- Neo4j（图数据库，可选，若启用 KG 功能）
- LanceDB（向量数据库，单文件，随 Python 进程部署）

---

## 方案对比

### 方案 A：Vercel + Railway（PaaS 托管）

```
Next.js ──────────────── Vercel (Hobby/Pro)
Python FastAPI ─────────── Railway (Starter)
MongoDB ─────────────────── MongoDB Atlas (已有)
DuckDB / LanceDB ────────── Railway 持久磁盘
Neo4j ───────────────────── Railway / Neo4j Aura Free
```

**月成本估算**：

| 服务 | 免费层限制 | 超出收费 | 家庭使用预估 |
|---|---|---|---|
| Vercel Hobby | 100GB 带宽, 100k 函数调用/天 | $0.15/GB | $0/月 |
| Railway Starter | $5 免费额度/月 | 按用量 | $0–5/月 |
| MongoDB Atlas Free | 512MB 存储 | M10: $57/月 | $0/月（够用） |
| Neo4j Aura Free | 200k 节点 | AuraDB Pro: $65/月 | $0/月（够用） |
| **合计** | — | — | **$0–5/月** |

**Pros**：
- 零运维：平台处理 SSL、扩容、故障恢复
- 一键部署（GitHub 连接自动 CI/CD）
- Railway 支持持久磁盘（DuckDB/LanceDB 文件不会丢失）

**Cons**：
- Railway Starter 5$/月额度耗尽后按量计费
- Python FastAPI 冷启动（Railway 在无流量时会 suspend，首次响应约 5-10s）
- Vercel Functions 有 10s 执行时间限制（AI 信号生成可能超时，需用 Railway 直接暴露 API）

---

### 方案 B：Docker + VPS

```
docker-compose.yml
├── nextjs:3000
├── python-api:8000
├── neo4j:7474,7687 (可选)
└── nginx:80,443 (反向代理 + SSL)
```

部署到 VPS（DigitalOcean / 腾讯云轻量应用服务器 / 阿里云 ECS）。

**月成本估算**：

| 服务商 | 规格 | 月费 | 备注 |
|---|---|---|---|
| DigitalOcean Droplet | 2vCPU / 4GB RAM | $24/月 | 有 100GB SSD |
| DigitalOcean Droplet | 1vCPU / 2GB RAM | $12/月 | 内存偏紧（Neo4j 需 2GB+） |
| 腾讯云轻量服务器 | 2vCPU / 4GB RAM | ¥45/月（~$6.5）| 国内访问快 |
| 阿里云 ECS | 2vCPU / 4GB RAM | ¥60/月（~$8.5）| 国内访问快 |
| **最低配（不含 Neo4j）** | 1vCPU / 2GB RAM | **$5–6/月** | DuckDB + LanceDB 够用 |

**是否靠谱**：✅ 完全靠谱，业内成熟方案。

- `next.config.ts` 设置 `output: 'standalone'` → `next build` 产出标准 Node.js 服务（`server.js`）
- Python FastAPI 用 `uvicorn api.main:app` 启动，无特殊依赖
- Nginx 配置示例：

```nginx
server {
    listen 443 ssl;
    server_name yourdomain.com;
    
    location / {
        proxy_pass http://localhost:3000;  # Next.js
    }
    
    location /api/ {
        # 代理由 Next.js /api/cn/* 内部处理，无需直接暴露 Python
    }
}
```

- SSL：`certbot --nginx -d yourdomain.com` 免费申请 Let's Encrypt
- CI/CD：GitHub Actions → SSH 到 VPS → `docker compose pull && docker compose up -d`

**Pros**：
- 固定月费，无意外账单
- 完全控制，可自定义任何配置
- Next.js 无执行时间限制（非 Serverless）

**Cons**：
- 需自己配置 Nginx、SSL、防火墙
- VPS 宕机需手动重启（可配置 `restart: always` Docker 策略）
- 不如 PaaS 弹性

---

### 方案 C：Raspberry Pi 自托管（家庭局域网）

```
树莓派 4B 8GB
├── Docker
│   ├── nextjs:3000
│   ├── python-api:8000
│   └── neo4j:7474 (可选，需 8GB RAM)
└── 家庭路由器 (局域网访问)
    └── Cloudflare Tunnel (可选，公网访问)
```

**可行性分析**：

| 指标 | 评估 | 说明 |
|---|---|---|
| **内存** | ✅ 8GB 够用 | Next.js ~300MB + Python FastAPI ~200MB + Neo4j ~500MB = ~1GB，余量充足 |
| **CPU** | ✅ 日常够用 | Cortex-A72 (ARM64) 4核；AI 信号生成较慢（Claude API 调用为主，本地 CPU 不是瓶颈） |
| **ARM64 兼容性** | ✅ 完全支持 | Node.js、Python、Neo4j、DuckDB 均有 ARM64 支持；Docker 镜像需指定 `platform: linux/arm64` |
| **存储** | ⚠️ 建议 SSD | MicroSD 读写寿命有限；DuckDB + LanceDB 频繁写入建议接 USB3 SSD |
| **耗电** | ✅ 极低 | 约 5–8W 满载；月电费约 ¥4–6（$0.5–1） |
| **可靠性** | ⚠️ 家用级 | 无 UPS 时断电可能损坏 DuckDB；建议加不间断电源或定期备份 |
| **公网访问** | 可选 | Cloudflare Tunnel 免费方案：在树莓派上运行 `cloudflared` 隧道，无需公网 IP，无需端口映射 |
| **月费用** | **~$0** | 一次性购买约 $80（4B 8GB）；电费 <$1/月 |

**推荐配置**：

```yaml
# docker-compose.yml (树莓派版本)
version: "3.9"
services:
  nextjs:
    image: openstock-nextjs:latest
    platform: linux/arm64
    restart: always
    ports: ["3000:3000"]
    
  python-api:
    image: openstock-python:latest
    platform: linux/arm64
    restart: always
    ports: ["8000:8000"]
    volumes:
      - ./data:/app/data  # DuckDB + LanceDB 持久化
    
  neo4j:  # 可选，若不用 KG 功能可注释掉
    image: neo4j:5-community
    platform: linux/arm64
    restart: always
    environment:
      NEO4J_AUTH: neo4j/yourpassword
      NEO4J_HEAP_INITIAL_SIZE: "256m"
      NEO4J_HEAP_MAX_SIZE: "512m"  # 树莓派内存节省
```

---

### 方案 D：Vercel + Render（全免费层）

- Next.js → Vercel Hobby（免费）
- Python → Render Free（免费，但有冷启动 50s+ ❗）

**月成本**：$0

**主要问题**：Render 免费层无流量时 15 分钟后休眠，冷启动需要 50-60 秒，不可接受。

**推荐指数**：⭐（仅适合功能演示）

---

## Vercel "天价账单"风险详细分析

> 用户原文：「如果你正在开发一个高流量或对成本敏感的项目……方案 C（Docker + VPS）固定费用 $5-$10，流量管够」

**结论：对于家庭个人使用，Vercel 免费层几乎不会产生费用，天价账单风险极低。**

具体分析：

| 触发条件 | Vercel Hobby 限制 | 家庭使用实际情况 |
|---|---|---|
| 带宽超限 | 100GB/月，超出 $0.15/GB | 个人使用远低于 1GB/月 |
| 函数调用超限 | 100,000 次/天 | 个人使用 <1,000 次/天 |
| 函数执行时间超限 | 10s/次 | ⚠️ AI 信号生成可能超时（通过 job-id 轮询避免，见 REQ-007） |
| 并发超限 | 无（Hobby 有软限制） | 个人使用无压力 |

**AI 信号调用的正确方式**：

AI 信号生成（`/cn/signals` → Python LangGraph）响应时间可能 30-60s，超过 Vercel 函数 10s 限制。解决方案：

- **选定方案：job-id 轮询**（REQ-007）：`POST /api/cn/analysis/trigger` 立即返回 `{ job_id }`，客户端每 5s 轮询 `GET /api/cn/analysis/status/{job_id}`，全程走 Next.js 代理，不绕过 better-auth/JWT
- ~~方案 1（已否决）~~：客户端直接调用 Python 后端——会绕过 better-auth 鉴权（见 ADR-001 REQ-003）
- 备选：SSE 流式响应——可行但 Vercel Streaming 有额外约束，job-id 轮询更简单

**推荐决策路径**：

```
开发阶段
  └── 本地 Mac：docker-compose up（所有服务本地运行）

功能验证阶段（上线初期）
  └── Vercel Free + Railway Free Starter
  └── 成本：$0–5/月

稳定使用阶段
  ├── 选项1（家庭）：树莓派 4B 8GB + docker-compose
  │   └── 成本：$0/月（一次性 $80 购机）+ Cloudflare Tunnel 免费
  └── 选项2（需要稳定性）：腾讯云/阿里云轻量服务器 2核4GB
      └── 成本：¥45–60/月（~$6-8）
```

---

## 决策

**主要部署目标**：Docker + 树莓派（家庭使用）

**上线初期过渡方案**：Vercel Free + Railway Free Starter，待功能稳定后迁移树莓派。

**关键配置**：

1. `next.config.ts` 添加 `output: 'standalone'`
2. 多阶段 Dockerfile（Next.js + Python 各自独立镜像）
3. `docker-compose.yml` 统一编排
4. GitHub Actions CI/CD（build → push to registry → deploy）
5. Cloudflare Tunnel（可选，外网访问无需公网 IP）

---

## 后果

- ✅ 树莓派方案：零月费，家庭使用完全可行
- ✅ 固定成本，无意外账单风险
- ✅ `docker compose up` 一键启动所有服务（开发和生产统一体验）
- ⚠️ 树莓派单点故障：建议开启 Docker `restart: always` + 定期 cron 备份 DuckDB 文件
- ⚠️ ARM64 镜像需在 Mac（x86/ARM）上交叉编译，`buildx` 支持，但首次构建较慢
