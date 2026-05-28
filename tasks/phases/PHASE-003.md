---
phase_id: PHASE-003
title: 深度分析 + AI 信号页
status: draft
priority: P1
---

## Goal

交付深度分析页（K 线图 + 技术指标 + 资金流）和 AI 信号页（LangGraph 信号触发 + 展示）。这两个页面引入 Plotly.js，需特别处理 SSR 兼容。

## In Scope

- REQ-006：深度分析页（`/cn/analysis`），含 OHLCV K 线、MA/MACD/RSI 指标、资金流向
- REQ-007：AI 信号页（`/cn/signals`），含触发分析按钮、信号列表、评分雷达图

## Out of Scope

- 调仓页（PHASE-004）
- Plotly 以外的图表优化

## Exit Criteria

1. `/cn/analysis` 加载后显示任意持仓股票的 K 线图（Plotly），数据从 Python 后端 `/api/cn/stock/{code}/ohlcv` 获取
2. `/cn/signals` 点击「触发分析」后，页面展示 AI 信号列表（含 composite_score 和 action_code）
3. Plotly 组件通过 `next/dynamic({ ssr: false })` 加载，首屏 JS 不包含 Plotly（network tab 验证）
4. AI 信号触发请求允许最长 60s 响应（非 Vercel Serverless 路径）

## Dependencies

- PHASE-002 完成

## Notes

AI 信号生成可能耗时 30-60s（LangGraph 多步 agent）。为避免 Vercel Functions 的 10s 超时，同时不绕过 better-auth/JWT 设计，信号触发采用 **job-id 轮询**模式（见 REQ-007）：

```
POST /api/cn/analysis/trigger  → Python 立即返回 { job_id }（<1s）
GET  /api/cn/analysis/status/{job_id}  → 客户端每 5s 轮询状态
```

全程走 Next.js 代理层（`/api/cn/*`），不直连 Python，不需要 `NEXT_PUBLIC_PYTHON_BACKEND_URL`。
