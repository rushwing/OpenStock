# ADR-001：统一平台策略 — OpenStock + my-invest-global 合并

- **状态**：已采纳
- **日期**：2026-05-29
- **决策者**：Daniel Wong

---

## 背景

当前存在两个独立项目：

| 项目 | 技术栈 | 市场 | 核心能力 |
|---|---|---|---|
| **OpenStock** | Next.js 15.5 + MongoDB + better-auth | 美股 | Watchlist、价格提醒、情绪分析、TradingView 图表、Inngest 邮件自动化 |
| **my-invest-global** | Python FastAPI + Vite/React + DuckDB + Neo4j + LanceDB | 大A（沪深） | 持仓管理、宏观红绿灯、LangGraph + Claude AI 信号、知识图谱、RAG 年报检索、Plotly 图表 |

**目标**：合并为一个统一平台，同时支持美股和大A，共享 Auth 和 UI 框架，避免维护两套独立前端。

**约束**：
- my-invest-global 的 Python AI 引擎（LangGraph/Neo4j/LanceDB/DuckDB）没有成熟的 TypeScript 对等生态，重写成本极高
- OpenStock 的 Inngest 邮件自动化、better-auth、TradingView 集成价值已验证，不应丢弃
- 以个人/家庭使用为主，部署成本敏感

---

## 可选方案

### 方案 A：TypeScript 全量重写

将 Python 后端所有功能用 TypeScript 重写为 Next.js API Routes。

| Pros | Cons |
|---|---|
| 单一技术栈，Vercel 一站式部署 | LangGraph 无 TS 版；Neo4j + LanceDB TS SDK 成熟度远低于 Python |
| 无跨语言 API 调用 | 需重写完整 AI 引擎（~3,000 行 Python），工期 > 3 个月 |
| 类型安全一致 | DuckDB 时序查询迁移成本高 |

**推荐指数**：⭐（工程量与收益严重不匹配）

---

### 方案 B：扩展 Vite 前端，废弃 OpenStock

以 my-invest-global 的 Vite 前端为主，加入美股功能，OpenStock 停止维护。

| Pros | Cons |
|---|---|
| Python 侧零修改 | 放弃 OpenStock 已有的 better-auth、Inngest 邮件自动化、TradingView 集成 |
| Vite 构建速度快 | Vite + React SPA 无 SSR，不利于 SEO |
| 前端仍是 React，可复用组件 | 需在 Vite 中重新实现 MongoDB Auth |

**推荐指数**：⭐⭐（丢弃 OpenStock 的已有投资）

---

### 方案 C：Next.js 统一前端（**选定**）

**保留 Python 后端不动**，用 OpenStock 的 Next.js 替换 Vite 前端。在 Next.js 中新增 `/api/cn/[...path]` 代理层，透明转发到 Python FastAPI 后端。

```
用户浏览器
     │
     ▼
Next.js (OpenStock 扩展)
  ├── /         → 美股功能（现有，不变）
  ├── /cn/*     → 大A功能（新增页面）
  └── /api/cn/* → 代理 → Python FastAPI
                           │
                    DuckDB + Neo4j + LanceDB
                    LangGraph + Tushare
```

| Pros | Cons |
|---|---|
| Python AI 引擎零修改，直接复用 | 需维护两个运行进程（Next.js + Python） |
| Next.js SSR 优化初始加载和 SEO | 部署需两个服务（但成本仍低） |
| 共享 Auth、导航、TradingView 组件 | 跨语言 API 通信（但通过代理层隐藏） |
| Vite 前端组件（纯 React）可直接移植到 Next.js | 需要 `/api/cn` 代理层维护 JWT 刷新逻辑 |
| Inngest 邮件系统可扩展为大A周报 | — |

**推荐指数**：⭐⭐⭐⭐⭐

---

### 方案 D：iframe 嵌套

Next.js 主站用 `<iframe>` 嵌入 Vite 前端。

| Pros | Cons |
|---|---|
| 改动最小 | Auth 无法跨 iframe 共享（cookie 域限制） |
| — | UX 割裂，移动端体验差 |
| — | SEO 不可索引 iframe 内容 |

**推荐指数**：⭐（不可行）

---

## 决策

**选定方案 C：Next.js 统一前端 + Python 后端代理**

核心判断：
- Python 生态在 AI/量化分析领域的优势（LangGraph/pandas/DuckDB）在短期内无法被 TypeScript 替代
- 代理层的复杂度远低于重写 AI 引擎的复杂度
- React 组件在 Vite 和 Next.js 之间的迁移成本极低（JSX 几乎可以直接复制）

---

## 美股 ↔ 大A 无缝切换设计

### URL 结构

```
/                   → 美股 Dashboard
/watchlist          → 美股 Watchlist
/stocks/[symbol]    → 美股个股详情（Finnhub + TradingView）

/cn/portfolio       → 大A 持仓总览（Tab1 Holdings）
/cn/analysis        → 大A 深度分析（Tab3 DeepAnalysis）
/cn/signals         → 大A AI 信号（Tab4 Signals）
/cn/rebalance       → 大A 调仓（Tab5 Rebalance）
```

URL 前缀 `/cn` 即为市场上下文标识，无需额外状态管理。

### 共用模块

| 模块 | 复用方式 |
|---|---|
| **Auth**（better-auth + MongoDB） | US 和 CN 共享同一套用户体系和会话 |
| **Header / Nav** | 统一组件，通过 `usePathname()` 高亮当前市场；导航栏分 US/CN 两组 |
| **TradingViewWidget** | 同一组件，通过 props 切换：US 传 `NYSE:AAPL`，CN 传 `SSE:600519` |
| **搜索（SearchCommand）** | 根据当前市场上下文调用不同数据源：US → Finnhub，CN → `/api/cn/stock/search` |
| **Toast / 通知** | 共用 `sonner` 库 |
| **Inngest 邮件** | US 已有；CN 可扩展大A周报（复用同一套 Inngest 函数结构） |

### 独立模块（不共享）

| 模块 | US | CN |
|---|---|---|
| 图表引擎 | TradingView embed（零依赖） | Plotly.js（react-plotly.js） |
| 价格数据 | Finnhub REST API | Tushare/AkShare via Python |
| 持仓数据 | MongoDB Watchlist | Python YAML + DuckDB |
| AI 分析 | Adanos 情绪 API | LangGraph + Claude（本地 Python）|
| 新闻 | Finnhub 英文新闻 | Python 爬取/Tushare 中文新闻 |

### 新增依赖（前端）

```json
"@tanstack/react-query": "^5.x",
"react-plotly.js": "^2.x",
"plotly.js": "^2.x"
```

Plotly 体积约 3MB，通过 `next/dynamic` + `{ ssr: false }` 懒加载，不影响首屏。

---

## Trade-off 总结

1. **两进程 vs 单进程**：双进程（Next.js + Python）增加了运维复杂度，但这个复杂度只在部署时显现，不影响日常开发。本地开发用 `docker-compose up` 一行命令即可启动所有服务。

2. **代理层延迟**：`/api/cn/[...path]` 在 Next.js 服务端转发请求到 Python，会增加约 1-5ms 的网络跳转延迟。这对于金融数据查询（通常 100ms+ 延迟）可以忽略。

3. **React Query vs 无状态 SSR**：CN 页面采用客户端 React Query（`staleTime=5min`），而非 Next.js Server Components。原因：Python 后端数据多为计算密集型（AI 信号生成），适合客户端缓存而非每次 SSR 重新请求。

4. **JWT 管理**：Python 后端使用 passphrase → JWT 方案。Next.js 服务端持有 passphrase（仅在 env var 中），自动为客户端请求附加 JWT，客户端无需感知。JWT 失效时服务端自动刷新。

---

## 后果

- ✅ Python AI 引擎完整保留，无迁移风险
- ✅ OpenStock 现有功能（Auth/Inngest/TradingView）100% 保留
- ✅ Vite 前端 React 组件约 70% 可直接移植（无需重写）
- ✅ 用户单次登录，美股/大A无缝切换
- ⚠️ 两个进程的部署需要配置文档（见 ADR-002）
- ⚠️ Plotly 引入 ~3MB JS bundle，需 `next/dynamic` 懒加载
