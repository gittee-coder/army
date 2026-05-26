---
title: 红线规则集（PreToolUse hook 源 spec）
type: runbook
status: active
tags: [security, hook, red-line, governance]
---

# 红线规则集

> 这是控制面的 **runtime-无关 spec**，编译产物是 `bin/army-redline-hook`（worker 经 `hooks/worker-settings.json` 的 `--settings` 注入）。
> **硬约束 > 软约束**：RED 用硬编码 regex 在工具执行前拦，不靠 LLM"记得别这么干"。LLM 判断是软约束（可能不遵守/被绕过/幻觉）；确定性闸门"数学上拦得住"。

## 上线策略（两档）
- **LOG-ONLY（默认，第一周）**：命中只写 `log/redline.log` + `runs.jsonl`，永远放行。先观察有无误报——一个误判会卡死整条流水线。
- **ENFORCE（`ARMY_REDLINE_ENFORCE=1`）**：命中 RED 输出 `permissionDecision=deny`，确定性拦截。`army-run-worker` 在无人值守上线后已默认 export 此变量。

## RED — 永远人 ack（硬拦，已编码进 hook）

| 规则 | 判定 |
|---|---|
| `git push` 到 main/master | Bash 命令含 `git push` + `main\|master`，且不是推 `army/*` 分支 |
| `git force-push`（必审批） | Bash 命令含 `git push … --force / --force-with-lease / -f` |
| 改 `.env` / 密钥凭据文件 | Edit/Write/Bash 命中 `.env`（放行 `.env.example/sample/template/dist`）/ `secrets.*` / `credentials*` / `*.pem` / `id_rsa\|ed25519\|ecdsa` / `.netrc/.npmrc/.git-credentials/.pypirc` / `*.p12/.pfx/.keystore` |
| 删 prod 资源（`rm -rf` 危险路径） | Bash 命中 `rm -rf … (prod\|production\|/var/www\|/etc)` |
| ≥10MB 视频/二进制提进站点仓 | Bash 含 `git add` + 站点仓目录名（`ARMY_SITE_REPO`）+ `.mp4/.mov/.webm/.zip/.tar/.gz/.psd/.wav` |

> 只编码**确定性可由命令文本判定**的规则。不可由文本确定的（如某外部服务的重试节流、合规备案缺漏）留给 PR 门 + 人审兜底，不进 hook。

## 容错原则
- hook 读不到事件 / 自身异常 → **放行**（exit 0），绝不因守卫故障卡死流水线。
- 规则判定抛异常 → 该条按未命中处理（`_safe`）。

## 命中后的行为
- LOG-ONLY：写 log + ledger（`role=redline, result=warn`），放行。
- ENFORCE：写 log + ledger（`result=deny`）+ 输出 deny 决策。LGTM/审批一类的"最后一拍"在 Auditor → IM 审批卡片处闭合（见 `docs/orchestration-loop.md`）。
