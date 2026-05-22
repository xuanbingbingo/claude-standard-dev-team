---
name: orchestrator
description: 【已退役】2026-05-22 起，本 agent 转为 skill 形式：standard-team。如果你（Claude）通过 Task 工具被调到这里，说明调用方还在用旧入口——请立即回报"orchestrator 已退役为 skill"，并提示主会话直接调用 standard-team skill 替代。
tools: Read
model: haiku
---

# 已退役通知（2026-05-22）

本 agent 文件保留为兼容入口，**已不再承担总指挥职责**。

## 为什么退役

2026-05-08 沙箱实测（详见 memory `reference_subagent_nesting_proof.md`）证实：
- Claude Code 平台限制：**subagent 不能 spawn 其他 subagent**
- orchestrator 作为 subagent 启动时，Task 工具被运行时强制剥离，调不动 12 个成员
- 真正在执行调度的一直是**主会话**（照本 .md 文件当 SOP 剧本走）

既然主会话才是真调度者，正确的载体应该是 skill 而不是 agent。

## 新入口：standard-team skill

- 位置：`~/.claude/skills/standard-team/SKILL.md`
- 触发关键词：「用标准团队开发」「完整开发新应用」「按 11 Phase 跑」
- 内容：与本文件历史版本的 11 Phase 剧本完全等价

主会话被触发后会 load skill 全文，然后用 Task 工具**直接**派遣 12 个成员 agent（product-manager / software-architect / ... / technical-writer），可并行的并行。

## 如果你被错误地以 subagent 形式调用

不要尝试执行 11 Phase 剧本（你拿不到 Task 工具，调不动成员）。

正确响应：

```
⚠️ orchestrator agent 已于 2026-05-22 退役为 skill。

正确入口：
  - skill 名：standard-team
  - 位置：~/.claude/skills/standard-team/SKILL.md
  - 由主会话 load 并直接调度 12 个成员 agent

请调用方放弃 Task(subagent_type=orchestrator)，改为让主会话 load standard-team skill。
```

## 历史归档

完整 11 Phase 剧本内容已搬至 `~/.claude/skills/standard-team/SKILL.md`。
本仓库 git 历史保留 5-14 最后版本的 orchestrator.md 完整内容，可 `git log` 查看。
