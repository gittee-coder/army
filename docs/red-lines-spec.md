---
title: 红线规则集（PreToolUse hook 源 spec）
type: runbook
project: _global
status: active
owner: pyf
created: 2026-05-22
tags: [security, hook, red-line, governance]
---

# 红线规则集

> 这是控制面"件 A"的 **runtime-无关 spec**。B（Mac mini 宿主）落地后，把本表编译成军团 Claude Code 的 `settings.json` PreToolUse hook 部署。
> **硬约束 > 软约束**：RED 用硬编码 regex 在工具执行前拦，不靠 LLM"记得别这么干"。LLM 判断是软约束（可能不遵守/被绕过/幻觉）；确定性闸门"数学上拦得住"。

## 上线策略
**第一周只 log 不拦**，确认无误报再切真拦——一个误判会卡死整条流水线。

## RED — 永远人 ack（硬拦）

| 规则 | 来源 |
|---|---|
| `git push` 到 main/master（force-push 必审批） | feedback_github_ops |
| 改 `.env` / secret / 删 prod 资源 | — |
| ≥10MB 视频/二进制提进 marginalia | feedback_pages_no_video |
| `bili-upload` 601 后重试（必须浏览器手动激活） | feedback_bili_upload |
| `tv/` 跳过 AIGC 水印 + 备案 | tv 合规默认开 |

## ORANGE — 走确认

| 规则 | 来源 |
|---|---|
| 部署到 book 服务器 | — |

## 预算闸

| 规则 | 来源 |
|---|---|
| `video-dreamina` 失败后自动重试抽卡（一次过为默认，不主动抽卡） | feedback_video_budget |

## 编译目标（B 落地后填）
- `settings.json` → `hooks.PreToolUse`，matcher 按工具名 + 命令 regex。
- 命中 RED → block + 推 Telegram 审批卡片；命中 ORANGE → 走确认；预算闸 → block。
- 第一周：命中只写 ledger（`runs.jsonl`）+ log，不 block。
