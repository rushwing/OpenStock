---
phase_id: PHASE-004
title: 调仓 + 导航整合
status: draft
priority: P1
---

## Goal

交付调仓页，并完成全站导航整合：US 和 CN 板块在同一导航栏中可一键切换，共享 Auth 和品牌视觉。

## In Scope

- REQ-008：调仓页（`/cn/rebalance`），含仓位偏差仪表盘、调仓建议列表
- REQ-009：导航 & 路由更新（`NAV_ITEMS` 分组，美股/大A分区，市场切换入口）
- REQ-010：大A常量 & TradingView 配置（SSE/SZSE 示例 symbol，热门A股列表）

## Out of Scope

- Docker 配置（PHASE-005）

## Exit Criteria

1. `/cn/rebalance` 显示白马股/弹性股当前仓位 vs 目标仓位的偏差，建议操作可见
2. Header 导航栏中 US 和 CN 板块分组清晰，当前区域高亮正确
3. 搜索（SearchCommand）在 CN 页面下调用 Python 后端搜索接口
4. `pnpm typecheck && pnpm test && pnpm lint` 通过

## Dependencies

- PHASE-003 完成

## Notes

导航分组方式：使用颜色区分（US 组用现有 teal 风格，CN 组用暖橙 accent，与 my-invest-global 的 tokens 保持一致，避免视觉混淆）。
