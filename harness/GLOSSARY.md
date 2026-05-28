---
harness_id: GLOSSARY-001
component: harness
owner: human-001
version: "0.1"
status: active
---

# Glossary

## §1 Agent Registry

| UID | Role | Tool |
|---|---|---|
| `optimizer-001` | Feature design, TC review, TC impl, Feature impl | Claude (claude-sonnet-4-6) |
| `evaluator-001` | Req review, TC design, TC impl review, Code review | Claude separate session / Codex |
| `human-001` | Scope approval, final merge, escalation resolution | Daniel |

## §2 Status Enum

| Value | Meaning | Actor |
|---|---|---|
| `draft` | Idea sketched, not ready for agent work | human-001 |
| `req_review` | Requirement design loop (optimizer-001 ↔ evaluator-001) | optimizer-001 or evaluator-001 |
| `tc_design` | Evaluator writes TC text files | evaluator-001 |
| `tc_review` | TC text review loop | optimizer-001 or evaluator-001 |
| `tc_impl` | Optimizer codes test cases | optimizer-001 |
| `tc_impl_review` | Evaluator reviews test code | evaluator-001 |
| `req_impl` | Optimizer codes the requirement | optimizer-001 |
| `req_impl_review` | Evaluator reviews implementation on draft PR | evaluator-001 |
| `pr_draft` | Draft PR ready for human merge | human-001 |
| `done` | PR merged, all bugs closed | — |
| `blocked` | External dependency or escalation | unassigned |

## §3 State Machine Transition Table

| # | From | Actor | Event | To | Owner after |
|---|------|-------|-------|----|-------------|
| T01 | `draft` | human-001 | Approves scope | `req_review` | `optimizer-001` |
| T02 | `req_review` | optimizer-001 | Completes requirement design | `req_review` | `evaluator-001` |
| T03 | `req_review` | evaluator-001 | Approves | `tc_design` | `evaluator-001` |
| T04 | `req_review` | evaluator-001 | Requests changes | `req_review` | `optimizer-001` (+review_round) |
| T05 | `req_review` | evaluator-001 | Approves + tc_policy=exempt | `req_impl` | `optimizer-001` |
| T06 | `tc_design` | evaluator-001 | Completes TC text | `tc_review` | `optimizer-001` |
| T07 | `tc_review` | optimizer-001 | Approves TC | `tc_impl` | `optimizer-001` |
| T08 | `tc_review` | optimizer-001 | Requests changes | `tc_review` | `evaluator-001` (+review_round) |
| T09 | `tc_impl` | optimizer-001 | Completes TC code | `tc_impl_review` | `evaluator-001` |
| T10 | `tc_impl_review` | evaluator-001 | Approves TC code | `req_impl` | `optimizer-001` |
| T11 | `tc_impl_review` | evaluator-001 | Requests changes | `tc_impl` | `optimizer-001` (+review_round) |
| T12 | `req_impl` | optimizer-001 | Impl done + CI passes + opens draft PR | `req_impl_review` | `evaluator-001` |
| T13 | `req_impl_review` | evaluator-001 | Approves + `gh pr ready` | `pr_draft` | `human-001` |
| T14 | `req_impl_review` | evaluator-001 | Requests changes | `req_impl` | `optimizer-001` (+review_round) |
| T15 | `pr_draft` | human-001 | Merges PR | `done` | — |
| T16 | any | any | External blocker or review_round ≥ 3 | `blocked` | `unassigned` |
| T17 | `blocked` | human-001 | Blocker resolved | blocked_from_status | blocked_from_owner |

## §4 Priority Enum

| Value | Meaning |
|---|---|
| `P0` | Critical — blocks PHASE exit criteria |
| `P1` | High — must ship in current phase |
| `P2` | Medium — nice to have in current phase |
| `P3` | Low — can defer to next phase |

## §5 Scope Enum

| Value | Meaning |
|---|---|
| `frontend` | Next.js pages and React components only |
| `backend` | Next.js API routes only |
| `fullstack` | Both frontend and backend |
| `docs` | Documentation only |
| `harness` | Harness files only |

## §6 Project-Specific Terms

| Term | Definition |
|---|---|
| **CN** | 大A（沪深 A 股），对应 URL 前缀 `/cn/*` |
| **US** | 美股（NYSE + NASDAQ），对应 URL 前缀 `/` |
| **Python 后端** | my-invest-global 的 FastAPI 服务，在 port 8000 运行 |
| **代理层** | Next.js `/api/cn/[...path]` 路由，透明转发到 Python 后端 |
| **CN JWT** | Python 后端颁发的 JWT，由 Next.js 服务端持有，客户端不可见 |
| **Holdings** | 持仓数据，存储在 Python 后端的 YAML + DuckDB 中 |
| **Signals** | AI 投资信号，由 LangGraph + Claude 生成 |
| **Macro State** | 宏观红绿灯（green/yellow/red），影响仓位目标 |
| **KG** | 知识图谱（Neo4j），存储机构持仓、板块关系 |
| **RAG** | 检索增强生成（LanceDB），基于年报 PDF |
