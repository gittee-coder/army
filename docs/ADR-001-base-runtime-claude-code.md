---
title: 底座运行时 = Claude Code（骑 Claude 订阅）
type: adr
status: active
tags: [runtime, billing, claude-code]
---

# ADR-001 底座运行时 = Claude Code

## 状态
Active

## 背景
个人 AI 组织需选一个 agent 运行时当底座。候选：Claude Code（Anthropic 官方 CLI）vs 第三方 agent 运行时（功能接近、已有部署）。原设计把"复用现成第三方运行时 vs Claude Code 从头拼，哪个省功"列为待评估。

## 决策
**底座 = Claude Code，登录 Claude 订阅跑。** 第三方运行时降为"可选 IM 网关 / 技能宿主"层，不当底座。

## 决定性理由（计费，一票否决）
- **第三方运行时多数只吃 BYOK（API key），喂不进 Claude 订阅。** 拿它当底座 = 全军 API 计费常驻（量大时月成本可观）。
- 操作者**主力在 Claude 订阅上**。换 API 成本爆炸，换其它模型性能不足。
- Claude Code 是官方 CLI，headless `claude -p` 的所有 worker 挂同一订阅 auth，**边际美元成本 ≈ 0**（订阅本来就在付）。

## 次要理由（治理机制原生）
- PreToolUse hook = 硬红线所需的确定性闸门（这正是 `army-redline-hook` 落地的接口）。
- subagent + worktree 最接近 PM/Worker/Auditor 的权责分离（SoD）。

## 引入的新约束
订阅是**用量上限**（滚动窗口 + 周封顶），非按 token 计费 → 代价从"美元账单"变成"峰值被限流"。
- 早 briefing + 几个 bug/周 + Auditor 定时巡检的节奏大概率塞得下。
- 全域 always-on + 多 worker 连轴才可能撞周限。
- **模型 = per-task 旋钮 + 角色默认**（Opus/Sonnet/Haiku 自由切换：`claude -p --model`）。质量优先：PM=Opus、Worker 默认 Opus（琐碎活降 Sonnet）、Auditor PR review=Sonnet、巡检/日报=Haiku。per-task 降档 = 订阅内泄压阀。

## 影响
- 宿主层：headless Claude Code 挂 systemd（或 launchd），登录订阅。
- 第三方运行时的 daemon/cron/通道若复用，只在"额外 IM 网关"层面按需评估，不影响底座。
