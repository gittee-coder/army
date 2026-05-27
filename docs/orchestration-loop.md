---
title: 编排循环 — 最小闭环 spec
type: spec
status: active
tags: [orchestration, pm, worker, auditor, state-machine]
---

# 编排循环 — 最小闭环 spec

把所有设计决策拼成一张实现级状态机。**最小闭环 = 软件部门**：操作者拍一个 bug → PM 派 code worker → 开 PR → Auditor review → 人在 IM 审批合并。全程过 guard / ledger / 红线 hook。

> 当前状态：最小闭环已端到端实跑（多个真实 bug 自主修复并合并）。dispatcher 已转 LIVE（systemd timer 15min 轮询），红线 hook 已切 ENFORCE，IM 双向桥常驻，每日 briefing（9:00）/日报（22:00）已挂 systemd timer。media/research 画像、多 worker 并行、briefing 深加工 = 后续。

---

## 1. 角色与运行位置（已定）

| 角色 | cwd | 身份来源 | 模型 | 写权 |
|---|---|---|---|---|
| **PM** | `~/army/` | 自动加载 `~/army/CLAUDE.md` + 注入 `roles/pm.md` | Opus | Issue/label/ledger/briefing |
| **Worker** | 项目 worktree | 项目自身 CLAUDE.md（cwd 自动）+ 注入 `roles/worker.md` | Opus 默认 | 仅自己的 feature 分支 + PR |
| **Auditor** | 项目目录 | 注入 `roles/auditor.md` | Sonnet（review）/ Haiku（巡检·日报） | **无写权**（只评论 PR / 写 reports） |

**常驻的只有 dispatcher（poller）+ IM bridge**；PM/Worker/Auditor 都是按触发唤起、干完即弃的事件式进程。

---

## 2. 任务状态机（事实源 = GitHub Projects v2 `status` 字段 + label）

```
          操作者开 Issue / PM 从 IM 代开
                       │
                       ▼
                  ┌─ triage ─┐   PM：读 + 决定接不接
       不接/缺信息 │          │ 接 + 拆解（plan 写进 Issue 评论）+ 打 army/triaged
            ┌──────┘          └──────┐
            ▼                        ▼
        blocked ◀──── 缺输入 ──────  ready   （进串行队列，按 repo 排序）
            ▲                        │ PM 派工（army-run-worker：建 worktree + 起 worker）
            │                        ▼
            │                   in-progress   worker：改码/测/开 PR（head=army/issue-N）
            │   worker 失败 3× ──────┤          包装层记 ledger（含 files_touched）
            │                        ▼
            │                     review      Auditor：审 + 安全扫 + VERDICT；打 army/audited
            │   VERDICT=CHANGES ────┤          ├─ CHANGES → IM 通知打回 + 回 in-progress
            │                        │ LGTM    └─ LGTM → ↓
            │                        ▼
       (RED 命中)             awaiting-approval   army-approve-card 推 IM 审批卡片
            │                        │ 人点批准
            └──── RED 拦截 ──────────┤ 人点拒绝 → 回 in-progress / blocked
                                     ▼
                                   done        IM bridge 跑 gh pr merge --squash + 记 ledger
```

### Label / 字段约定
- 入口 label（操作者打）：`auto-eligible`（军团可接）、可选 `red-line`、`model:opus|sonnet`
- 路由 label：`project:<name>`（对应 `projects.yaml`）
- 流转 label：`army/triaged`（PM 已分诊）、`army/audited`（Auditor 已审，防重复触发）
- PR ↔ Issue：PR body 写 `Closes #N`，head 分支 `army/issue-N`，合并即自动关 Issue

---

## 3. 各角色的进入/退出契约

### 3.1 PM（事件式：WAKE→REHYDRATE→ACT→PERSIST→RELEASE）

**触发**：① IM `@PM …` ping；② poller 发现 `auto-eligible` 且未 `army/triaged` 的新 Issue；③ cron 9:00 briefing；④ worker 回报需 PM 决策。

**REHYDRATE（只捞这次要的）**：`gh issue view N`、`projects.yaml` 对应条目、相关 PRD（按需 `cat`）。**不读全历史。**

**ACT**：
- 分诊：能接 → 拆解 plan 写进 Issue 评论 + 打 `army/triaged`；不能接/缺信息 → `blocked` + IM 问操作者。
- 派工（`ready` → `in-progress`）：调 `army-run-worker <proj> <issue> [model]`。默认串行。
- 验收：worker 开 PR 后**不自己合**，由 dispatcher 另行唤 Auditor。

**PERSIST（commit 点）**：更新 Issue 评论 + label + `army-ledger --role pm ...`。

**RELEASE**：session 结束。同群 burst 内 follow-up 才 `--resume`（RESUME_TTL=1h，对齐 prompt cache TTL）。

### 3.2 Worker（一任务/一 repo 串行簇 一 session）

**触发**：`army-run-worker`（已在 worktree 里）。
**做**（`worker_type=code`）：在 `army/issue-N` 分支改 → 跑该项目测试（`projects.yaml` 的 `test`）→ `gh pr create`（body `Closes #N`）。
**ledger**：code worker **不自记**，`army-run-worker` 包装层在结束后记一行（wall-clock + rc + `files_touched`）。media/research 暂无包装层，需自记。
**红线**：绝不 push main/master；触红线由 PreToolUse hook 拦（见 §6）。
**失败**：测试/构建失败自修 ≤3 次，仍不过 → `result=fail` + `blocked` + 回报 PM。
**退出**：PR 开了即结束。

### 3.3 Auditor（只读，SoD）

**触发**：poller 发现开放 PR（head=`army/*`）且未 `army/audited`。
**做**：审 diff + 安全扫 → 评论结论 + 末行输出 `VERDICT=LGTM|CHANGES` + 打 `army/audited` 标。
  - LGTM → `army-approve-card` 推 IM 审批卡片，等人点批准。
  - CHANGES → IM 通知打回 + 回 `in-progress`（worker 修）。
**绝不**：合并代码（合并是人的审批动作）、改业务代码（allowlist 无 Edit/Write）。

---

## 4. 常驻件：dispatcher（poller）+ IM bridge

全部 `army-guard || exit 0` 开头。

### 4.1 IM bridge（feishu-bridge，WebSocket 长连接，实时）
- 收 `@PM …` → 唤起 headless PM（流式 stream-json → 实时刷新一张流式卡片当回执）。维护 `chat→session-id` 映射：RESUME_TTL 内同群 follow-up 走 `--resume` 吃缓存。
- 收审批卡片回调（`{action: merge|diff|reject, repo, pr, issue}`）：`merge` → `gh pr merge --squash --delete-branch`；`diff` → 回传 diff；`reject` → 评论打回。
- 凭据走 env 文件（`~/.army/im.env`），脚本无硬编码 ID/secret。

### 4.2 poller（dispatcher，systemd timer 15min，兜底网）
- `gh issue list --label auto-eligible`：未 `army/triaged` 的新 Issue → `army-run-pm`。
- `gh pr list`：head=`army/*` 且未 `army/audited` 的 PR → `army-run-auditor`。
- 默认 DRY（只查不跑）；`ARMY_DISPATCH_LIVE=1` 真唤起。
- 末尾常驻调用 `army-dashboard-sync`（按日历天节流，把作战看板同步到站点仓）。

### 4.3 串行队列
- 默认单一在飞 worker（concurrency=1）；`ready` 任务按 `project` 聚类入队。

### 4.4 两道找 PM 的门（同一角色）
- **远程/异步**：IM bridge（dispatcher 唤起 headless PM）。人不在时的主通道。
- **本地/直连**：`~/army/bin/pm`（在 `~/army` 起交互 Claude + 注入 `roles/pm.md`）。人在终端时的控制台。
- 两门是**同一个 PM 角色**——都从持久层（Issue/Projects/知识库/ledger）水合、看同一世界；只是触发方式不同，不共享 live session 也无妨。

---

## 5. 幂等 & 断点续（宿主会关机）

- **状态在 GitHub 不在内存**：dispatcher 重启后从 `gh issue/pr list` + label **重建队列**，不靠记忆。
- **每个动作前查现态**：PM 跳过已 `army/triaged` 的 Issue；Auditor 跳过已 `army/audited` 的 PR；worker 若该 Issue 已有开放 PR 则不重复开。
- **session-id 映射丢失无害**：下次唤起重新水合。
- **briefings/reports** 幂等键 = 日期+类型，存在即更新。
- **worktree 残留**：合并后清理已合并分支的 worktree（`git worktree prune` / PM rehydrate 时清）。
- **真 7×24**：systemd `Linger=yes` + 宿主电源不睡 + 开机自启（catch-up 补关机期间漏的）。

---

## 6. 接线：guard / ledger / 红线 hook

- **总开关**：每个 systemd 服务 + 每次唤起脚本第一行 `~/army/bin/army-guard || exit 0`。
- **账本**：worker 由包装层记、PM/Auditor 收尾各调 `~/army/bin/army-ledger`；红线 hook 命中也记一行。
- **红线**：worker 的 Claude Code 经 `--settings hooks/worker-settings.json` 配 PreToolUse hook（`army-redline-hook`，编译自 `red-lines-spec.md`）。已切 **ENFORCE**：命中 RED 确定性 deny。RED 的"最后一拍人审"在 Auditor LGTM → IM 审批卡片处闭合。

---

## 7. 最小闭环验收

1. 在某个已注册的软件项目开一个 `auto-eligible` 的 P2/P3 bug Issue。
2. **零人工干预**（除最后审批一拍）走完：PM 接单+拆解 → worker 开 PR → Auditor LGTM → IM 卡片 → 人批准 → 合并 + 关 Issue。
3. `runs.jsonl` 有完整链路（pm triage/dispatch + worker（含 files_touched）+ auditor + pm 审批合并），外加红线 hook 演练行。

---

## 8. 落地顺序（实现，已全部走通）
1. dispatcher 骨架（systemd 服务 + guard）+ poller
2. PM 唤起 + 分诊 + 派 1 个 code worker（worktree）+ 开 PR
3. Auditor 唤起 + review → VERDICT → awaiting-approval
4. IM bridge：@PM ping（流式卡片）+ 审批卡片 merge/diff/reject
5. 红线 hook：LOG-ONLY → ENFORCE
6. dispatcher 转 LIVE + systemd 常驻（Linger + 不睡 + 开机自启）
