---
title: 编排循环 — 最小闭环 spec
type: spec
project: _global
status: active
owner: pyf
created: 2026-05-22
tags: [orchestration, pm, worker, auditor, state-machine]
---

# 编排循环 — 最小闭环 spec

把前面所有设计决策拼成一张实现级状态机。**最小闭环 = Phase 1 软件部门**：pyf 拍一个 bug → PM 派 code worker → 开 PR → Auditor review → pyf Telegram 审批合并。全程过 guard / ledger / 红线。

> 依据：`AI-军团自建方案.md` §3/§4/§5/§6/§7。本 spec 只覆盖最小闭环；media/research 画像、多 worker 并行、briefing 自动化深加工、图书管理员 = 后续。

---

## 1. 角色与运行位置（已定）

| 角色 | cwd | 身份来源 | 模型 | 写权 |
|---|---|---|---|---|
| **PM** | `~/army/` | 自动加载 `~/army/CLAUDE.md` + 注入 `roles/pm.md` | Opus | Issue/label/看板/ledger/briefing |
| **Worker** | 项目 worktree | 项目自身 CLAUDE.md（cwd 自动）+ 注入 `roles/worker.md` | Opus 默认 | 仅自己的 feature 分支 + PR |
| **Auditor** | 项目目录 | 注入 `roles/auditor.md` | Sonnet（review）/ Haiku（巡检·日报） | **无写权**（只评论 PR / 写 reports） |

**常驻的只有 dispatcher**（含 telegram-bridge + poller + 串行队列）；PM/Worker/Auditor 都是按触发唤起、干完即弃的事件式进程。

---

## 2. 任务状态机（事实源 = GitHub Projects v2 `status` 字段）

```
          pyf 开 Issue / PM 从 Telegram 代开
                       │
                       ▼
                  ┌─ triage ─┐   PM：读 + 决定接不接
       不接/缺信息 │          │ 接 + 拆解（plan 写进 Issue 评论）
            ┌──────┘          └──────┐
            ▼                        ▼
        blocked ◀──── 缺 pyf 输入 ── ready   （进串行队列，按 repo 排序）
            ▲                        │ PM 派工（建 worktree + 起 worker）
            │                        ▼
            │                   in-progress   worker：改码/测/开 PR + 记 ledger
            │   worker 失败 3× ──────┤
            │                        ▼
            │                     review      Auditor：审 + 安全扫
            │   review 不过 ────────┤          ├─ 不过 → 评论 + 回 in-progress（worker 修）
            │                        │ 过      └─ 过 → ↓
            │                        ▼
       (RED 命中)             awaiting-approval   推 Telegram 审批卡片给 pyf
            │                        │ pyf 批准
            └──── RED card ──────────┤ pyf 拒 → 回 in-progress / blocked
                                     ▼
                                   done        合并 PR + 关 Issue + 记 ledger
```

### Label / 字段约定
- 入口 label（pyf 打）：`auto-eligible`（军团可接）、可选 `red-line`、`model:opus|sonnet`
- 路由 label：`project:<name>`（必填，对应 `projects.yaml`）
- Project v2 `status` 字段 = 上图状态（单一事实源）；`risk`、`project` 为辅助字段
- PR ↔ Issue：PR body 写 `Closes #N`，合并即自动关 Issue

---

## 3. 各角色的进入/退出契约

### 3.1 PM（事件式：WAKE→REHYDRATE→ACT→PERSIST→RELEASE）

**触发**：① Telegram `@PM …` ping；② poller 发现 `auto-eligible` 且 `status=triage` 的新 Issue；③ cron 9:00 briefing；④ Auditor/worker 回报需 PM 决策。

**REHYDRATE（只捞这次要的）**：`gh issue view N`、相关 Project 卡、`projects.yaml` 对应条目、相关 PRD（按需 `cat`）。**不读全历史。**

**ACT**：
- 分诊：能接 → 拆解，把 plan（步骤 + 验收标准 + 选定 `model`）写进 Issue 评论；不能接/缺信息 → `blocked` + Telegram 问 pyf。
- 派工（`ready` → `in-progress`）：见 §4 dispatch 命令。默认串行：只在队列空时取下一条；队列按 repo 聚类。
- 验收：worker 开 PR 后不自己合，转 Auditor。

**PERSIST（commit 点）**：更新 Issue 评论 + Project status + `army-ledger --role pm ...`。

**RELEASE**：session 结束。对话 burst 内的 follow-up 才 `--resume`。

### 3.2 Worker（一任务/一 repo 串行簇 一 session）

**触发**：PM dispatch（已在 worktree 里）。
**读**：Issue plan（作为任务输入）+ cwd 的项目 CLAUDE.md。
**做**（`worker_type=code`）：在 `army/issue-N` 分支改 → 跑该项目测试（`projects.yaml` 的 `test`）→ `gh pr create`（body `Closes #N`）→ `army-ledger --role worker --issue N --files ... --result ok|fail`。
**红线**：绝不 push main/master；触红线由 PreToolUse hook 拦（见 §6）。
**失败**：测试/构建失败自修 ≤3 次，仍不过 → `army-ledger --result fail` + `status=blocked` + 回报 PM。
**退出**：PR 开了即结束。

### 3.3 Auditor（只读，SoD）

**触发**：poller 发现 `status=review` 的开放 PR。
**做**：审 diff + 安全扫 → 通过 → 评论 LGTM + `status=awaiting-approval` + dispatcher 推 Telegram 审批卡片；不过 → 评论问题 + 回 `in-progress`（worker 修）。
**绝不**：合并代码（合并是 pyf 的审批动作）、改业务代码。

---

## 4. Dispatcher（唯一常驻件）+ 具体命令

轻量守护进程（systemd 服务），三个子职责，全部 `army-guard || exit 0` 开头：

### 4.1 telegram-bridge（getUpdates 长轮询，实时）
- 收 `@PM …` → 唤起 PM（带消息为触发上下文）。维护 `chat→session-id` 映射：同线程 10min 内 follow-up 走 `--resume`，超时丢弃。
- 收审批卡片回调：`[批准合并]` → `gh pr merge --squash`；`[看 diff]` → 回传 diff；`[拒绝]` → 回 `in-progress`/`blocked`。`ack <issue>` 关键字兜底。

### 4.2 poller（cron 15min，兜底网）
- `gh issue list --label auto-eligible`：`status=triage` 的新 Issue → 唤起 PM。
- `gh pr list`：`status=review` 的 PR → 唤起 Auditor。
- 开机后先跑一次（catch-up，补 WSL 关机期间漏的）。

### 4.3 串行队列
- 维护单一在飞 worker（concurrency=1）；`ready` 任务按 `project` 聚类入队；只有 PM 显式标"独立又急"才临时放第 2 个。

### 唤起命令（实现级）
```bash
# PM（cwd=~/army 自动加载 CLAUDE.md）
cd ~/army && claude -p --append-system-prompt "$(cat ~/army/roles/pm.md)" \
  --model opus "触发：<telegram msg | new issue #N>"

# 派 Worker：dispatcher 先建 worktree，再起 worker
path=$(yq ".projects.<proj>.path" ~/army/projects.yaml)   # 解析项目路径
git -C "$path" worktree add "$HOME/army/worktrees/<proj>/issue-N" -b army/issue-N
cd "$HOME/army/worktrees/<proj>/issue-N" && \
  claude -p --append-system-prompt "$(cat ~/army/roles/worker.md)" \
  --model "${MODEL:-opus}" "任务 #N：<plan 摘要>。完成开 PR（Closes #N），记 ledger。"

# Auditor
cd "$path" && claude -p --append-system-prompt "$(cat ~/army/roles/auditor.md)" \
  --model sonnet "review PR #M（issue #N）：审 diff + 安全扫，结论评论回 PR。"
```

---

### 4.4 两道找 PM 的门（同一角色）
- **远程/异步**：Telegram（dispatcher 唤起 headless PM）。人不在时的主通道。
- **本地/直连**：`~/army/bin/pm`（在 `~/army` 起交互 Claude + 注入 `roles/pm.md`）。人在终端时的控制台。
- 两门是**同一个 PM 角色**——都从持久层（Issue/Projects/pyf-docs/ledger）水合、看同一世界；只是触发方式不同，不共享 live session 也无妨（状态在文件不在聊天历史）。

## 5. 幂等 & 断点续（WSL 会关机）

- **状态在 GitHub 不在内存**：dispatcher 重启后从 `gh issue/pr list` + Project status **重建队列**，不靠记忆。
- **每个动作前查现态**：PM 跳过已过 triage 的 Issue；worker 若该 Issue 已有开放 PR 则不重复开。
- **session-id 映射丢失无害**：下次唤起重新水合。
- **briefings/reports** 幂等键 = 日期+类型，存在即更新。
- **worktree 残留**：dispatcher 启动时清理已合并分支的 worktree（`git worktree prune`）。

---

## 6. 接线：guard / ledger / 红线 hook

- **总开关**：每个 systemd 服务 + 每次唤起脚本第一行 `~/army/bin/army-guard || exit 0`。
- **账本**：worker/PM/Auditor 收尾各调 `~/army/bin/army-ledger`。
- **红线**：worker 的 Claude Code 配 PreToolUse hook（编译自 `runbooks/red-lines.md`）。**第一周 log-only**：命中只 `army-ledger` 记一行 + log，不 block；确认无误报再切真拦。RED 命中真拦后 → Telegram RED 卡片等 pyf ack。

---

## 7. 最小闭环验收（对齐 §9 Phase 1）

1. 在某个已注册的软件项目（如 cue）开一个 `auto-eligible` 的 P2/P3 bug Issue。
2. **零人工干预**（除最后审批一拍）走完：PM 接单+拆解 → worker 开 PR → Auditor LGTM → Telegram 卡片 → pyf 批准 → 合并 + 关 Issue。
3. `runs.jsonl` 有完整 3 行（pm/worker/auditor）。
4. 一周跑通 3-5 个，PM 决策准确率 >80%。

---

## 8. 最小闭环【不含】（后续）
media/research worker 画像 · 多 worker 并行 · briefing/日报自动化深加工 · 图书管理员 · Projects 自动流转细化 · 红线真拦（先 log-only） · Windows 自启/不睡（live 后补）。

---

## 9. 落地顺序（实现）
1. dispatcher 骨架（systemd 服务 + guard）+ poller（先只 PM 唤起）
2. PM 唤起 + 分诊 + 派 1 个 code worker（worktree）+ 开 PR
3. Auditor 唤起 + review → awaiting-approval
4. telegram-bridge：审批卡片 + `@PM` ping
5. 红线 hook log-only 挂上
6. 跑验收（§7）
