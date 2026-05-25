# army — 用 Claude Code 搭 PM/Worker/Auditor 三 agent 自动开发流水线

> 约 300 行 Bash + Python，**没有 Airflow、没有消息队列、没有自建编排框架**，靠 Claude Code 的 headless 模式（`claude -p`）把 `Issue → 分诊 → 写码测试 → 开 PR → review → 人审 → 合并` 串成一条 **agent 间零人工干预**的流水线。
>
> _A ~300-line multi-agent auto-dev pipeline on top of Claude Code: PM triages, Worker codes in an isolated worktree, Auditor reviews read-only, you approve the merge. State lives in GitHub + files; no orchestration framework._

核心判断：**多 agent 自动开发的难点不在编排框架，在三件事——职责怎么分权、状态放哪、安全闸怎么确定性化。** 这三样都能用文件系统 + GitHub + 几个脚本表达，不需要重型基础设施。

---

## 它跑通过什么（真实证据）

在一个前端项目里开一个布局 bug、打上 `auto-eligible` 标，然后没再碰键盘，直到最后那一下「批准合并」：

```
Issue#1 ──worker(Opus,125s)──▶ PR#2 ──auditor(Sonnet,90s)──▶ LGTM ──人批准──▶ merged
```

`runs.jsonl` 执行账本里这一单的真实记录（见 [`runs.example.jsonl`](./runs.example.jsonl)）：

```jsonl
{"role": "worker",  "issue": 1, "duration_s": 125.0, "result": "ok"}
{"role": "auditor", "issue": 1, "duration_s": 90.0,  "result": "ok"}
{"role": "pm", "task": "...already resolved — PR#2 merged...no re-dispatch...", "issue": 1, "result": "ok"}
```

最后那行是亮点：对一个**已合并的** Issue 再跑 PM，它核对 GitHub + 账本两个事实源后判断「已解决、再派会重复」，直接 no-op——**幂等**是无人值守的前提。

---

## 架构：三段触发 + 一份注册表，没有常驻框架

```
┌─────────────────────────────────────────────────────────────┐
│  事实源（状态全在这里，不在任何进程的内存里）                  │
│   GitHub Issues + Projects   ←── 任务/状态（label 路由）      │
│   GitHub PR                  ←── 代码评审载体                │
│   projects.yaml              ←── 项目注册表（路径/repo/测试） │
│   runs.jsonl                 ←── 执行账本                    │
└───────────────────────────┬─────────────────────────────────┘
                            │  每次唤起都从这里"水合"
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
   army-run-pm        army-run-worker     army-run-auditor
   (Opus, 只读)        (Opus, 可写自分支)   (Sonnet, 只读)
        └─ 干完即弃：没有常驻 agent；进程跑完就没了，状态早写回事实源
```

三个角色不靠共享内存/消息传递协作，而是靠**同一份外置事实源**：Worker 开了 PR 写在 GitHub 上，Auditor 下次被唤起去 GitHub 读到这个 PR 就知道审什么。

## 核心：职责分离是结构保证，不是 prompt

让同一个 agent 既写码又审自己的码，本质是让它自己制衡自己——做不到。所以 SoD（职责分离）**落在 `--allowedTools` 白名单上，不是 prompt 里的客套话**：

```bash
# Worker：给 Edit/Write
claude -p ... --permission-mode acceptEdits --allowedTools Bash Edit Write Read Grep Glob

# Auditor：没有 Edit/Write —— 从机制上保证它改不了代码
claude -p ... --permission-mode default    --allowedTools Bash Read Grep Glob
```

> 能用白名单表达的约束，就不要用 prompt 表达。白名单是确定性闸门，prompt 是概率性请求。

模型分层：PM=Opus（判断准），Worker 默认 Opus（琐碎活降 Sonnet），Auditor=Sonnet。所有 headless 进程挂同一份 Claude 订阅 auth，边际成本≈0（详见 [`docs/ADR-001`](./docs/ADR-001-base-runtime-claude-code.md)）。

---

## 快速开始

前置：[Claude Code](https://claude.com/claude-code)（登录可跑 headless `claude -p`）、[`gh`](https://cli.github.com/) CLI（已 `gh auth login`）、`python3` + `pyyaml`。

```bash
git clone <this-repo> ~/army && cd ~/army
cp projects.example.yaml projects.yaml      # 填你的项目：path / repo / test
chmod +x bin/*

# 在某 repo 开一个 Issue，打 auto-eligible 标，然后手动触发一单全闭环：
bin/army-run-pm   <project> <issue>         # PM 分诊 + 派 worker + 开 PR
bin/army-run-auditor <project> <pr> <issue> # Auditor 只读 review + 评论
# 看 PR、满意就在 GitHub 上点合并（合并永远是人来点）

bin/army-dispatch                           # poller（默认 DRY 只查不跑）
ARMY_DISPATCH_LIVE=1 bin/army-dispatch      # 真自动唤起 PM

echo PAUSE > ~/.army-state                  # 总开关：一条命令停全军
```

> 脚本默认以 `~/army` 为家。`gh` 用其自身认证；org 仓如需 org-scoped token，运行前 `export GH_TOKEN=...`。

## 目录

```
bin/        8 个脚本（共 ~300 行）：dispatch / run-{pm,worker,auditor} / guard / ledger / watchdog / pm
roles/      PM / Worker / Auditor 三份角色手册（--append-system-prompt 注入）
CLAUDE.md   组织宪法（PM 运行于 ~/army 时自动加载）—— 这是作者的真实版，按你的红线改
docs/       ADR-001（为何选 Claude Code 当底座）/ red-lines-spec（红线 hook 源 spec）/ orchestration-loop（完整状态机 spec）
projects.example.yaml  项目注册表示例
runs.example.jsonl     执行账本样例（上面那一单的真实记录）
```

## 安全：硬约束 > 软约束

1. **工具白名单**（确定性，已落地）：Auditor/PM 没有 Edit/Write，结构上改不了代码。
2. **总开关**（确定性，已落地）：每个脚本第一行 `army-guard || exit 0`，`PAUSE` 一键停全军。
3. **红线 hook**（规划中）：把 [`docs/red-lines-spec.md`](./docs/red-lines-spec.md) 编译成 Claude Code 的 PreToolUse hook（push main / 改 secret / 删 prod 等硬拦），**先 log-only 一周**再切真拦。

## 当前状态（诚实）

**能跑的**：三角色在真实 bug 上端到端验证过、PR 已合并；白名单 SoD、git worktree 隔离、账本、总开关都在用；PM 幂等 no-op 已验证。

**还没做完**：dispatcher 默认仍 DRY（要 `ARMY_DISPATCH_LIVE=1`）；Auditor 自动触发、红线 hard hook、Telegram 审批桥、systemd 常驻**都还没接**。换句话说：**手动触发能跑通全闭环，但还不是 24h 无人值守。** 别一上来追无人值守——先让「手动触发也能跑通」稳定下来。

## License

MIT — 见 [LICENSE](./LICENSE)。
