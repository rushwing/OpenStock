---
harness_id: REQ-STD-001
component: requirement-management
owner: human-001
version: "0.1"
status: active
---

# Requirement Standard（OpenStock TypeScript 适配版）

> 从 my-invest-global 移植，CI 命令已适配 TypeScript/pnpm 项目。

## §0 Mandatory Pre-Work Protocol — HARD STOP

**Before writing any code, document, or task artifact for a requirement, an agent MUST verify all three conditions below. If any condition fails, stop immediately and report to Daniel. Do not write anything.**

### Check table

| # | Condition | How to check | Failure action |
|---|-----------|-------------|----------------|
| C1 | REQ file exists | `ls tasks/features/REQ-NNN.md` | Stop. Ask Daniel for the correct REQ ID. |
| C2 | `owner` field matches you | `grep "^owner:" tasks/features/REQ-NNN.md` | Stop. The REQ belongs to another agent. Do not proceed. |
| C3 | `status` is a valid work state for your role | See table below | Stop. The REQ is not ready for your action. Report the status. |

### Valid work states per agent

| Agent | Valid `status` values | Action at each state |
|-------|-----------------------|----------------------|
| **optimizer-001** | `req_review` | Design or revise the requirement text |
| **optimizer-001** | `tc_review` | Review TC text written by evaluator-001 |
| **optimizer-001** | `tc_impl` | Implement test cases (code) |
| **optimizer-001** | `req_impl` | Implement the requirement (code) |
| **evaluator-001** | `req_review` | Review the requirement; approve or request changes |
| **evaluator-001** | `tc_design` | Write TC text files in `tasks/test-cases/` |
| **evaluator-001** | `tc_impl_review` | Review TC code written by optimizer-001 |
| **evaluator-001** | `req_impl_review` | Review implementation; convert draft PR on approval |
| **human-001** | `pr_draft` | Review and merge the ready PR |

---

## §1 Scope

Applies to every feature or change delivered in OpenStock after the harness scaffold is complete (REQ-001 done). Governs REQ file format, state machine, agent handoff protocol, and blocking rules.

---

## §2 File Location and Naming

```
tasks/
  features/     REQ-NNN.md   (active)
  bugs/         BUG-NNN.md   (active)
  test-cases/   TC-NNN-SS.md (NNN = REQ digits, SS = sequence)
  archive/done/ REQ-NNN.md   (archived after done)
```

REQ IDs are sequential integers, zero-padded to 3 digits: `REQ-001`, `REQ-042`.

---

## §3 Frontmatter Schema

```yaml
---
req_id: REQ-001
title: "短标题"
status: draft
owner: unassigned
priority: P1                   # P0 (critical) | P1 | P2 | P3
phase: PHASE-001
scope: fullstack               # frontend | backend | fullstack | docs | harness
tc_policy: required            # required | optional | exempt
tc_exempt_reason: ""
depends_on: []
test_case_ref: []
acceptance: "single sentence verifiable criterion"
review_round: 0
pending_bugs: []
blocked_reason: ""
blocked_from_status: ""
blocked_from_owner: ""
pr_number: ""
---
```

---

## §4 State Machine（同 my-invest-global）

States: `draft` → `req_review` ↔ `tc_design` → `tc_review` ↔ `tc_impl` → `tc_impl_review` → `req_impl` → `req_impl_review` → `pr_draft` → `done`

See `harness/GLOSSARY.md` for full transition table.

---

## §5 CI Gates（TypeScript 适配）

**T12 要求 optimizer-001 在 req_impl 完成后运行：**

```bash
pnpm typecheck   # tsc --noEmit（等价于 mypy --strict）
pnpm test        # vitest run（等价于 uv run pytest）
pnpm lint        # eslint（等价于 ruff check）
```

全部通过后方可 `gh pr create --draft`。

**Lint 基线说明**：原始仓库存在约 83 个 lint 问题（36 errors / 47 warnings），均为 pre-existing。在 REQ-001 完成 lint 基线清理之前，lint gate 定义为：

> **PR 引入的新 lint 错误数量 = 0**（不要求清零 pre-existing 问题）

验证命令（对比分支与 main 的新增错误）：

```bash
# 仅检查本 PR 改动的文件是否引入新错误
git diff --name-only origin/main...HEAD | grep -E '\.(ts|tsx)$' | xargs pnpm eslint
```

REQ-001 完成后，gate 升级为全量 `pnpm lint` 零错误通过。

---

## §6 Acceptance Criterion Rules

- 一句话，现在时态，可验证
- 必须引用具体可观测行为（不写"works correctly"）
- Good: `访问 /api/cn/health 返回 HTTP 200 且 body 包含 { "status": "ok" }`
- Bad: `代理层正常工作`
