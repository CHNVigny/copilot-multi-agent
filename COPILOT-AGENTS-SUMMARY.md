# Copilot Multi-Agent 编程配置 — 完整总结文档 v6

> 基于 Anthropic Building Effective Agents + Teamday Complete Guide + Karpathy 4 原则 + 7 Design Patterns
> 详细记录了 11 个 Copilot Agent 的完整设计、触发条件、交互流程和安装方式。

---

## 一、项目概述

这是一个为 **VS Code + GitHub Copilot** 设计的多角色协作编程 Agent 团队配置。通过 `.agent.md` 文件定义 11 个专业化 Agent，由 `DevCoordinator` 自动编排完整的软件开发流水线。

### 核心设计原则（v6 精华）

1. **Evaluator-Optimizer 闭环**：Reviewer 给出行级 before/after 修复指令，Implementer 精准执行——不"打回去重写"
2. **Self-Reflection 循环**：Implementer 写完代码→自验证→自审查→产出验证→自我修正→再循环，最多 2 轮，把问题消灭在交付前
3. **Context Summary 机制**：每次调度子代理前注入精准状态摘要，解决"Agent 失忆"
4. **循环上限 + 升级机制**：任何子代理最多重试 3 轮，超过后整理"已做尝试 + 失败原因"向用户精确求助
5. **双重架构验证**：Architect 在 Phase 1 审查计划 + Phase 2 审查实现
6. **压力测试可执行**：StressTester v4 有 terminal 权限，实际运行压测命令
7. **模型分层**：规划/审查类用 Haiku/Flash 省钱，实现/审计类用 Sonnet/GPT-5 保质量

### v6 稳定性的三层防御

```
Layer 1: Implementer Self-Reflection → 自我发现 60-80% 的问题
Layer 2: Reviewer Evaluator-Optimizer → 精确修复指令，不模糊打回
Layer 3: Coordinator 升级机制 → 重试耗尽后精确求助，不瞎猜
```

---

## 二、11 个 Agent 完整清单

| # | Agent Name | 职责 | 触发条件 | 工具 | 模型 |
|---|---|---|---|---|---|
| 1 | **DevCoordinator** | 总指挥 v6，弹性流水线 + Context Summary + 升级机制 | 所有新任务 | agent, read, search, edit, terminal | Sonnet 4.6 / GPT-5 |
| 2 | **Planner** | 需求拆解为 Goal-Driven 任务列表 | Phase 1 | read, search | Haiku 4.5 / Flash |
| 3 | **Architect** | 双重架构审查（Phase 1 计划 + Phase 2 实现） | Phase 1 + Phase 2.5 | read, search, terminal | Haiku 4.5 / Flash |
| 4 | **ContextManager** | 为子代理生成精准 Context Summary | 每次调度前 | read, search, edit | Haiku 4.5 / Flash |
| 5 | **Implementer** | Self-Reflection 码匠，7 条自检表 + 产出验证 | Phase 2 | read, search, edit, terminal | Sonnet 4.6 / GPT-5 |
| 6 | **Debugger** | 诊断并修复编译/运行时 bug | 编译/测试失败 | read, search, edit, terminal | Sonnet 4.6 / GPT-5 |
| 7 | **Reviewer** | Evaluator-Optimizer，6 维打分 + 行级修复指令 | 每个 task 完成后 | read, search, terminal | Sonnet 4.6 / GPT-5 |
| 8 | **StressTester** | 魔鬼代言人 v4，理论+实际执行压测 | 复杂逻辑/并发 | read, search, terminal | Haiku 4.5 / Flash |
| 9 | **SecurityAuditor** | OWASP Top 10 + CWE Top 25 全覆盖 | 安全相关代码 | read, search, terminal | Sonnet 4.6 |
| 10 | **TestWriter** | 编写单元/集成测试 | Phase 5 | read, search, edit, terminal | Haiku 4.5 / Flash |
| 11 | **QATester** | 手动测试/Bug Report 生成 | Phase 5 | read, search, terminal | Haiku 4.5 / Flash |

---

## 三、完整流水线

```
Phase 1: 规划
  Planner 拆解 → Architect (Phase 1) 架构预审

Phase 2: 实现
  ContextManager 生成摘要 → Implementer (Self-Reflection 循环)
  → Reviewer (Evaluator-Optimizer: 打分 + 行级修复)
  → 循环直到 ≥ 7/10 或触发升级

Phase 2.5: 架构后审 (v6 新增)
  Architect (Phase 2) 审查实现是否偏离架构

Phase 3 ‖ Phase 4: 压力测试 + 安全审计 (并行)
  StressTester (v4, 实际执行压测) ‖ SecurityAuditor

Phase 5: 测试
  TestWriter → QATester

Phase 6: 收尾
  更新 progress.md + decisions.md + 文档产出验证
```

### 循环上限

| Agent | 最大重试 | 超限行为 |
|---|---|---|
| Implementer | 3 轮 | 🆘 升级给用户 |
| Planner | 2 轮 | 🆘 升级给用户 |
| 其他 | 2 轮 | 🆘 升级给用户 |

---

## 四、karpathy 4 原则（全局行为准则）

1. **Think Before Coding**：不确定时停下来，列出所有解读，不做猜测
2. **Simplicity First**：只写必要代码，200 行能搞定不写 1000 行
3. **Surgical Changes**：只改必须改的，不顺手优化，不清预先存在的死代码
4. **Goal-Driven Execution**：每个步骤必须有可验证的成功标准

---

## 五、Context Summary 机制

每次调度子代理前，ContextManager 生成一段 500 字以内的精准上下文：
- 项目技术栈快照
- 成功标准
- 涉及文件状态
- 架构约束（来自 decisions.md）
- 禁止事项（最高优先级）

原则："精确到刚好够"——过多上下文导致 confusion，过少导致盲猜。

---

## 六、升级机制（Escalation）

当子代理达到重试上限时，Coordinator 整理：

```
🆘 任务升级
- 原始需求
- 已尝试：轮次 | 内容 | 结果
- 阻塞点
- 建议方案：2-3 个供用户选择
```

**不自行尝试第 4 种方案。** 等待用户裁决。

---

## 七、文档产出强制验证

Phase 1 结束和 Phase 6 结束时，Coordinator 必须执行 terminal 命令验证文档存在：
- `ls -la docs/project/pipeline/`
- `test -s docs/project/sprints/current.md`
- `grep $(date) docs/project/memory/progress.md`
- `test -f docs/project/memory/decisions.md`

MISSING → 立即创建 → 重新验证 → 循环直到全 OK。

---

## 八、安装方式

将 `agents/` 目录下所有 `.agent.md` 文件复制到 VS Code Copilot 的 agents 目录：
- `.github/copilot-agents/`（推荐）
- 或使用 Copilot 自定义 Agent 路径

Agent 发现依赖 Coordiator 的 `agents:` frontmatter 声明。
