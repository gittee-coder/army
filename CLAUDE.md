# CLAUDE.md — AI 组织控制面（军团运行时之家）

> 控制面共享操作手册。**在 `~/army/` 下运行的 agent（主要是 PM）自动加载本文件。**
> 各角色的具体职责手册在 `roles/{pm,worker,auditor}.md`；dispatch 时用 `--append-system-prompt` 注入
> （worker 运行在项目目录、不在本树下，故靠注入而非本文件拿角色身份）。

## 你服务于谁
操作者（人）。契约：**默认放行、异常才找、RED 才打断**。人只做四件事——写文档 / 拍任务 / IM 审批 / 和 PM 聊。

## 事实源（记忆在文件里，不在聊天历史里）
- **任务/状态**：GitHub Issues + Projects v2（label 路由 `auto-eligible` / `red-line` / `project:*`）
- **知识库**：你的文档仓（PRD/决策/runbook；PM 读 PRD、写 briefing/reports）
- **执行账本**：`~/army/runs.jsonl`（每次跑完 `bin/army-ledger` 追加一行）
- **项目注册表**：`~/army/projects.yaml`（project → 路径/repo/部门/worker 画像/测试/worktree 基）

## 总开关 & 出口
- 每个任务/systemd 服务第一行：`~/army/bin/army-guard || exit 0`（PAUSE 时跳过）
- 出口探活：`~/army/bin/army-watchdog`（DOWN = 宿主侧代理挂了，全军断 API）

## 红线（硬约束，详见 `docs/red-lines-spec.md`）
- **RED（永远人 ack）**：push main/master、force-push、改 .env/secret、删 prod 资源、≥10MB 二进制提进站点仓
- 由 `bin/army-redline-hook` 做 PreToolUse 确定性拦截（worker 经 `hooks/worker-settings.json` 注入）
- 上线策略：第一周 hook 只 log 不拦（LOG-ONLY），确认无误报再 `ARMY_REDLINE_ENFORCE=1` 切真拦

## 工作路径铁律
- 只在【具体项目路径】里干活，别拿一堆项目并列的根目录当工作路径（不在那种根上跑 git）
- worker 永远 `--cwd <项目路径>`（自动加载该项目自己的 CLAUDE.md）

## 上下文经济
事件式、状态外置、Worker 默认串行吃 cache。干完即 PERSIST 写回事实源，session 可弃。
