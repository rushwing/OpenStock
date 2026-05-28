---
phase_id: PHASE-002
title: 大A数据层 + 持仓总览页
status: draft
priority: P1
---

## Goal

建立客户端数据请求基础设施（TanStack Query + CN hooks），并交付第一个可用的大A页面：持仓总览（`/cn/portfolio`）。

## In Scope

- REQ-004：TanStack Query Provider + CN 数据 hooks（从 my-invest-global `frontend/src/api/hooks.ts` 移植）
- REQ-005：持仓总览页（`/cn/portfolio`），含持仓表格、盈亏指标、宏观红绿灯、持仓饼图

## Out of Scope

- Plotly 图表（在 PHASE-003 随 analysis 页面引入）
- 调仓和信号页面

## Exit Criteria

1. 访问 `/cn/portfolio`，持仓列表从 Python 后端加载，显示正确的股票代码、成本价、现价、盈亏
2. 宏观红绿灯（绿/黄/红）正确渲染
3. React Query DevTools 显示请求命中缓存（5 分钟内第二次访问不发新请求）
4. `pnpm typecheck && pnpm test && pnpm lint` 通过

## Dependencies

- PHASE-001 完成（代理层可用）

## Notes

CN hooks 的 base URL 改为 `/api/cn`，移除 Zustand auth store 依赖（JWT 由代理层服务端持有）。组件从 Vite SPA 迁移到 Next.js `'use client'` 组件，语法基本不变。
