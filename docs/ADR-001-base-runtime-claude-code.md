---
title: 底座运行时 = Claude Code（骑 Claude Max 订阅）
type: adr
project: _global
status: active
owner: pyf
created: 2026-05-22
tags: [runtime, billing, openclaw, claude-code]
---

# ADR-001 底座运行时 = Claude Code

## 状态
Active（2026-05-22 定，关掉原"待评估"）

## 背景
个人 AI 组织需选一个 agent 运行时当底座。候选：Claude Code（Anthropic 官方 CLI）vs OpenClaw（已在 Mac mini 跑，功能最接近）。原设计文档把"龙虾当底座 vs Claude Code 从头拼，哪个省功"列为待评估。

## 决策
**底座 = Claude Code，登录 Claude Max 订阅跑。** OpenClaw 降为"可选 IM 网关 / 技能宿主"层，不当底座。

## 决定性理由（计费，一票否决）
- **OpenClaw 只吃 BYOK（API key），喂不进 Claude Max 订阅。** 拿它当底座 = 全军 API 计费常驻（~$150-250/月）。
- pyf **主力在 Claude Max 订阅上**。换 API 成本爆炸，换 DeepSeek 等其他模型性能不足。
- Claude Code 是官方 CLI，headless `claude -p` 的所有 worker 挂同一订阅 auth，**边际美元成本 ≈ 0**（订阅本来就在付）。

## 次要理由（治理机制原生）
- PreToolUse hook = 硬红线所需的确定性闸门（OpenClaw 的安全是 pairing/沙箱/allowlist，缺内容级 per-action 闸）。
- `/team` + subagent + worktree 最接近 PM/Worker/Auditor 的权责分离（SoD）。

## 引入的新约束
Max 是**用量上限**（5h 滚动 + 每周封顶），非按 token 计费 → 代价从"美元账单"变成"峰值被限流"。
- Phase 0/1 节奏（早 9 点 briefing + 几个 bug/周 + Auditor 定时巡检）大概率塞得下。
- 全域 always-on + 2 worker 连轴才可能撞周限。
- **模型 = per-task 旋钮 + 角色默认（Opus/Sonnet 自由切换：`claude -p --model` / subagent `model` 参数）**。质量优先：PM=Opus、Worker 默认 Opus（琐碎活降 Sonnet）、Auditor PR review=Sonnet 巡检/日报=Haiku。per-task 降档 = Max 内泄压阀；**当前不引入 API/本地溢出**（pyf 取舍）。

## 影响
- §8 宿主层：Mac mini 上 headless Claude Code 挂 launchd，登录 Max 订阅。
- OpenClaw 的 daemon/cron/通道若复用，只在"额外 IM 网关"层面按需评估，不影响底座。
