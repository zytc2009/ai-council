# Multi-AI Discussion — 实现任务清单（交互式向导版）

> **状态更新（2026-04-02）**：所有任务（Phase 0-5）已全部完成，包括 P1 优化项。

基于更新后的设计文档，实现无参数启动的交互式向导模式。

---

## Phase 0: CLI 检测模块 ✅

### Task 0.1 — 创建 `lib/cli_detector.py` ✅

**文件**：`lib/cli_detector.py`

---

### Task 0.2 — 创建 `lib/streaming_runner.py` ✅

**文件**：`lib/streaming_runner.py`

---

## Phase 1: 交互式向导框架 ✅

### Task 1.1 — 多行输入收集 ✅

### Task 1.2 — CLI 选择与确认 ✅

### Task 1.3 — 主持人选择 ✅

### Task 1.4 — 配置确认 ✅

---

## Phase 2: 实时输出集成 ✅

### Task 2.1 — Phase 1 实时输出改造 ✅

### Task 2.2 — Phase 2 实时输出改造 ✅

### Task 2.3 — Phase 3 实时输出改造 ✅

---

## Phase 3: 无参数启动入口 ✅

### Task 3.1 — 主入口改造 ✅

### Task 3.2 — 交互向导主流程 ✅

---

## Phase 4: 配置持久化（P1）✅

### Task 4.1 — 检测到的 CLI 自动保存 ✅

**功能**：
- `agent detect --save` 可手动触发保存
- `_run_interactive_wizard` 自动保存已安装的 CLI
- 修复了 Windows 路径在 bash 中的转义问题

---

## Phase 5: 回退机制（P1）✅

### Task 5.1 — 手动输入 CLI 路径 ✅

**功能**：
- 无 CLI 时自动进入手动配置流程
- 增加了 `{prompt_file}` 占位符强制校验
- 支持交互式验证命令可用性

---

## Phase 6: 上下文优化（P1）✅

### Task 6.1 — 讨论历史的上下文压缩 ✅

**文件**：`lib/discussion_orchestrator.py`, `lib/context.py`

**功能**：
- 集成 `compress_history` 到 `DiscussionOrchestrator`
- 当历史记录超过 4000 字符时自动调用 LLM 进行摘要压缩
- 保留最近 1 轮原文，更早轮次转为摘要

---

## 依赖关系与实施顺序

```
Phase 0 (基础模块) ──→ Phase 1 (交互输入) ──→ Phase 2 (实时输出) ──→ Phase 3 (入口集成) ──→ Phase 4-6 (P1 优化)
```

**✅ 全部完成（2026-04-02）**

---

## 文件变更总览

| 操作 | 文件 | Task |
|------|------|------|
| 新增 | `lib/cli_detector.py` | 0.1 |
| 新增 | `lib/streaming_runner.py` | 0.2 |
| 修改 | `lib/discussion_orchestrator.py` | 2.1, 2.2, 2.3, 6.1 |
| 修改 | `cli_assistant.py` | 1.1, 1.2, 1.3, 1.4, 3.1, 3.2, 4.1, 5.1 |
| 修改 | `lib/agent_runner.py` | 修复 Windows 路径转义 |
| 修改 | `lib/streaming_runner.py` | 修复 Windows 路径转义 |
| 不变 | `lib/config.py` | 无需修改 |
| 不变 | `lib/meeting.py` | 无需修改 |
