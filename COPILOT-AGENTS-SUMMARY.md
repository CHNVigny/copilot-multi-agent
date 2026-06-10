# Copilot Multi-Agent 编程配置 — 完整总结文档

> 本文件可供 OpenClaw 智能体或其他 AI 助手作为参考文档使用。
> 详细记录了 10 个 Copilot Agent 的完整设计、触发条件、交互流程和安装方式。

---

## 一、项目概述

这是一个为 **VS Code + GitHub Copilot** 设计的多角色协作编程 Agent 团队配置。通过 `.agent.md` 文件定义 10 个专业化 Agent，由 `DevCoordinator` 自动编排完整的软件开发流水线：规划 → 架构审查 → 代码实现 → 调试 → 代码审查 → 压力测试 → 安全审计 → 自动测试 → 手动 QA。

核心特性：
1. **Coordinator-Worker 编排模式**：主 Agent 通过 `agents:` frontmatter 声明子 Agent，使用 `runSubagent` 工具自动分派任务
2. **6 维度加权打分制审查**：正确性×3 + 安全性×2 + 性能×1.5 + 可维护性×1.5 + 一致性×1 + 健壮性×1
3. **决策置信度机制**：≥90 自动执行，70-89 标注风险，<70 暂停询问
4. **反馈循环**：实现→审查→修复→再审查，循环直到通过
5. **Tool 最小化**：每个 Agent 只配最小必要工具集，避免 Context Rot
6. **模型分层**：规划/审查类用 Haiku/Flash 省钱，实现/审计类用 Sonnet/GPT-5 保质量
7. **项目记忆持久化**：PROJECT_BRIEF + sprints + decisions + progress，4 文件跨 session 记忆

---

## 二、10 个 Agent 完整清单

以下是每个 Agent 的 name、职责、触发条件、工具权限、推荐模型：

| # | Agent Name | 职责 | 触发条件 | 工具 | 模型 |
|---|---|---|---|---|---|
| 1 | **DevCoordinator** | 总指挥，6 阶段流水线自动化编排 + 项目管理 | 所有新任务 | agent, read, search, edit, terminal | Sonnet 4.6 / GPT-5 |
| 2 | **Planner** | 需求拆解为可执行任务列表，输出结构化计划 | 每次任务 | read, search | Haiku 4.5 / Flash |
| 3 | **Architect** | 对照代码库审查计划，验证架构一致性，发现可复用组件 | 每次任务（与 Planner 可并行） | read, search | Haiku 4.5 / Flash |
| 4 | **Implementer** | 按计划编写代码，不做调试 | 每个实现任务 | read, search, edit | Sonnet 4.6 / GPT-5 |
| 5 | **Debugger** | 诊断编译/运行时错误，小修自己动手，大修交还 Implementer | 编译/类型检查/测试失败时 | read, search, edit, terminal | Sonnet 4.6 / GPT-5 |
| 6 | **Reviewer** | 6 维度加权打分审查，加权总分<7.0 驳回 | 每个任务实现完成后 | read, search, terminal | Sonnet 4.6 / GPT-5 |
| 7 | **StressTester** | 并发/边界/资源耗尽/降级/组合爆炸压力测试 | 复杂逻辑、并发、边界条件 | read, search | Haiku 4.5 / Flash |
| 8 | **SecurityAuditor** | OWASP Top 10 + CWE Top 25 全覆盖安全审计 | 用户输入、认证、权限、数据处理、文件上传 | read, search, terminal | Sonnet 4.6 |
| 9 | **TestWriter** | 单元测试 + 集成测试编写，自动运行验证 | 功能实现后 | read, search, edit, terminal | Haiku 4.5 / Flash |
| 10 | **QATester** | 手动/探索性测试，从用户视角验证功能，输出 Bug Report | 自动测试完成后 | read, search, terminal | Haiku 4.5 / Flash |

Agent 文件存放路径：

- 项目级：`.github/agents/*.agent.md`
- 用户全局（所有项目）：`%USERPROFILE%\.vscode\agents\` (Windows) 或 `~/.vscode/agents/` (Linux/macOS)

---

## 三、6 阶段开发流水线（DevCoordinator 自动编排）

### Phase 1: 规划
- 步骤 1: 调用 Planner 拆解需求为结构化任务列表（输出需包含：涉及文件、实现步骤、边界条件、依赖与顺序、风险点、验收标准）
- 步骤 2: 调用 Architect 对照代码库审查计划（检查：模式复用、架构一致性、依赖影响、可维护性）
- 步骤 3: Architect 发现问题 → 反馈 Planner 更新 → 再审查，循环直到通过
- 步骤 4: 更新 `docs/project/sprints/current.md`，将 Phase 1 标记为 ✅
- 步骤 1 前：创建 `docs/project/pipeline/YYYY-MM-DD-任务简称.md`
- 可并行：Planner 拆解的同时，Architect 预扫描代码库

### Phase 2: 实现
- 步骤 5: 按计划逐项分配任务给 Implementer 编码
- 步骤 6: 编译/类型检查失败 → 调 Debugger 诊断
- 步骤 7: 每个任务完成后立即调 Reviewer 审查
- 步骤 8: Reviewer 加权总分 < 7.0 → Implementer 修复 → 再审，循环直到 ≥ 7.0
- 步骤 5+7: 每次分发和每次审查结果记录到 pipeline 日志

### Phase 3: 压力测试（按需触发）
- 步骤 9: 复杂逻辑/并发/边界条件 → 调 StressTester（攻击维度：逻辑漏洞、并发竞态、资源耗尽、组合爆炸、降级恢复、边缘用户行为）
- 步骤 10: 发现潜在缺陷 → Implementer 加固 → 重新过 Reviewer
- 可并行：StressTester + SecurityAuditor 可同时运行

### Phase 4: 安全审计（按需触发）
- 步骤 11: 涉及用户输入/认证/权限/数据处理/文件上传 → 调 SecurityAuditor（OWASP A01-A10 + CWE Top 25）
- 步骤 12: 🚨 高危问题 → Implementer 立即修复 → 重新过 Reviewer + SecurityAuditor

### Phase 5: 测试
- 步骤 13: 调 TestWriter 写单元/集成测试（必须覆盖：happy path、分支路径、边界条件、异常输入、异步路径）
- 步骤 14: 测试失败 → Debugger 诊断 → Implementer 修复 → 重新跑测试
- 步骤 15: 调 QATester 做手动检查清单验证（功能正确性、用户体验、边界异常、跨功能交互）
- 步骤 16: QATester 发现问题 → 创建 Issue 记录 → Implementer 修复
- 可并行：TestWriter + QATester 可同时运行

### Phase 6: 收尾
- 步骤 20: 更新 `docs/project/memory/progress.md`
- 步骤 21: 若产生新架构决策 → 追加 `docs/project/memory/decisions.md`
- 步骤 22: 将 pipeline 日志所有阶段标记为 ✅，填入结束时间
- 步骤 23: 向用户汇报：做了什么、改了哪些文件、测试结果、待决事项
- 步骤 24: **附上完整的阶段追踪表格**

---

## 四、6 维度代码审查评分体系（Reviewer）

| 维度 | 权重 | 检查内容 |
|---|---|---|
| 1. 正确性 | ×3 | 逻辑正确性、边界条件、异步处理、类型安全 |
| 2. 安全性 | ×2 | 用户输入校验、注入风险、认证授权 |
| 3. 性能 | ×1.5 | 循环内 I/O、N+1 查询、大数据量处理 |
| 4. 可维护性 | ×1.5 | 命名表意性、DRY、注释质量 |
| 5. 一致性 | ×1 | 代码风格、架构决策遵循 |
| 6. 健壮性 | ×1 | 外部依赖降级、重试/超时/熔断 |

判定规则：
- 🔴 驳回：加权总分 < 7.0 或正确性/安全性任何维度 < 4
- 🟡 有条件通过：≥ 7.0 但有建议
- 🟢 通过：≥ 8.0 且无阻塞问题

---

## 五、决策置信度机制（DevCoordinator）

| 置信度 | 行为 |
|---|---|
| ≥ 90 | 直接执行，不询问用户 |
| 70-89 | 执行但标注「请确认：…」 |
| < 70 | 暂停，问一个精确问题 |

适用范围：
- 代码实现细节（命名、结构、库选择）：默认 ≥ 90，不询问
- 架构方向、技术选型、破坏性变更：显式评估置信度

---

## 六、项目记忆与流水线日志

### 目录结构

```
docs/project/
├── memory/                         # 项目记忆（长期、跨 session）
│   ├── PROJECT_BRIEF.md            # 项目概述、技术栈、架构决策
│   ├── decisions.md                # ADR 风格架构决策记录
│   └── progress.md                 # 每次对话后的进度摘要
├── sprints/                        # Sprint 管理
│   └── current.md                  # 当前 Sprint 任务列表、完成状态、阻塞项
└── pipeline/                       # 流水线执行日志（每次任务一份）
    └── YYYY-MM-DD-任务简称.md       # 任务级可追溯记录
```

### Coordinator 初始化检查清单

```
□ docs/project/memory/PROJECT_BRIEF.md  ← 若不存在，结束后创建
□ docs/project/memory/decisions.md      ← 若不存在，结束后创建
□ docs/project/memory/progress.md       ← 若不存在，结束后创建
□ docs/project/sprints/current.md       ← 若不存在，结束后创建
□ docs/project/pipeline/                ← 若不存在，立即 mkdir
```

### 流水线日志内容

每次任务一份日志，包含：
- 时间线（启动/完成时间）
- 任务拆解（Planner 输出）
- **子代理调用日志表**（时间/Agent/任务/状态/结果摘要）
- **阶段追踪表**（Phase 1-6，状态符号 ⬜⏳✅❌⏭️）
- Reviewer 分数

每完成一个阶段、每一次子代理调用、每一次 Review 结果，立即写入日志。每完成一个 Phase 向用户汇报当前阶段追踪表格。

Stale 检测：子代理分派后超过 3 轮没有返回结果，主动追踪。

---

## 七、各 Agent 关键约束清单

### DevCoordinator
- [ ] 每次只给子 Agent 一个明确小任务，保持上下文干净
- [ ] 子 Agent 返回后负责综合结果、判断是否需要循环
- [ ] 置信度 < 70 标注让用户拍板
- [ ] 禁止跳过任何阶段（除非用户明确说不需要）
- [ ] 每次完成 Phase 6 后更新项目记忆文件

### Planner
- [ ] 优先搜索代码库找可复用组件
- [ ] 标注需要 Architect 确认的架构决策点
- [ ] 需求不清晰时列出「待澄清」问题，不猜测
- [ ] 拆解粒度：1-3 文件、50-200 行代码
- [ ] 标注任务间依赖关系（Task N 依赖 Task M / 可并行项）

### Architect
- [ ] 检查代码库中已有类似实现
- [ ] 检查项目约定设计模式
- [ ] 检查 `docs/project/decisions.md` 中相关架构决策
- [ ] 用 terminal 跑 `tree`/`ls -R` 确认目录层级，跑依赖分析工具检查循环依赖
- [ ] 不写代码，只判断对错

### Implementer
- [ ] 严格执行计划，不自由发挥
- [ ] 先 read 再 edit，不盲改
- [ ] 参考 `docs/project/decisions.md` 不违反已有架构决策
- [ ] 不顺手重构无关代码
- [ ] 不改格式化
- [ ] 不擅自增删依赖
- [ ] 函数 < 50 行
- [ ] 公共函数有类型注解
- [ ] 不吞异常、不裸 throw
- [ ] 空值/undefined 显式处理
- [ ] 改完代码立刻用 terminal 跑编译/类型检查验证
- [ ] 遇到编译/运行时错误先自己尝试修复不超过 2 次
- [ ] 2 次后仍未解决 → 标注 `# NEEDS_DEBUG` 让 Coordinator 调 Debugger

### Debugger
- [ ] 只诊断，不重新实现
- [ ] 先复现错误再诊断
- [ ] 按「最近变更→依赖链→环境差异→并发时序」顺序排查
- [ ] 区分根因 vs 表象
- [ ] < 10 行小修自己动手，> 10 行大修交还 Implementer

### Reviewer
- [ ] 6 维度全打分，输出加权总分
- [ ] 正确性或安全性 < 4 → 直接驳回
- [ ] 加权总分 < 7.0 → 驳回
- [ ] 对每个变更文件用 read 读完整 diff

### StressTester
- [ ] 覆盖逻辑漏洞、并发竞态、资源耗尽、组合爆炸、降级恢复、边缘用户行为共 6 个攻击维度
- [ ] 输出抗压等级（💪坚固/😐一般/😰脆弱）
- [ ] 发现严重缺陷标注「阻塞/高/中/低」

### SecurityAuditor
- [ ] 全覆盖 OWASP Top 10（A01 到 A10）
- [ ] 额外覆盖 CWE Top 25（CWE-79/89/78/22/352/918/287/798）
- [ ] 运行 `npm audit` / `pip audit` / `cargo audit` 检查依赖
- [ ] 每个漏洞标注 CWE 编号 + OWASP 类别 + 攻击向量 + 修复方案
- [ ] 🚨 高危 = 立即阻塞
- [ ] 安全审计通过不代表 Reviewer 通过——两者独立
- [ ] 只做安全审计，不评价代码风格

### TestWriter
- [ ] 覆盖 happy path + 每个 if/else 分支（MC/DC）+ 边界条件 + 异常输入 + 异步路径
- [ ] 自动检测项目测试框架
- [ ] 测试相互独立、不依赖执行顺序
- [ ] 不修改业务代码
- [ ] 测试失败先分析是测试问题还是代码 bug，bug 标注 `# BUG:` 让 Coordinator 协调修复

### QATester
- [ ] 从用户视角验证（功能正确性、用户体验、边界异常、跨功能交互）
- [ ] 发现 Bug 输出结构化 Bug Report（复现步骤、预期/实际行为、严重程度 Blocker/Major/Minor）
- [ ] 测试完输出 QA 结论（签收/有条件签收/拒收）
- [ ] 绝不修代码
- [ ] 如果有 Blocker，Coordinator 分配给 Implementer 修复后重新 QA

---

## 八、工具最小化原则（Context Rot 防护）

每个 Agent 只配最小必要工具，避免上下文膨胀导致判断模糊：

| Agent | agent | read | search | edit | terminal |
|---|---|---|---|---|---|
| DevCoordinator | ✅ | ✅ | ✅ | ✅ | ✅ |
| Planner | — | ✅ | ✅ | — | — |
| Architect | — | ✅ | ✅ | — | ✅ |
| Implementer | — | ✅ | ✅ | ✅ | ✅ |
| Debugger | — | ✅ | ✅ | ✅ | ✅ |
| Reviewer | — | ✅ | ✅ | — | ✅ |
| StressTester | — | ✅ | ✅ | — | — |
| SecurityAuditor | — | ✅ | ✅ | — | ✅ |
| TestWriter | — | ✅ | ✅ | ✅ | ✅ |
| QATester | — | ✅ | ✅ | — | ✅ |

Planner 不需要 edit（只分析不写代码），StressTester 不需要 edit 和 terminal（纯逻辑分析），QATester 不需要 edit（不修代码）。

Architect 有 terminal 用于跑依赖分析/目录结构验证。Implementer 有 terminal 用于编译/类型检查自验证，遇到复杂错误再转交 Debugger。

---

## 九、模型分层策略

| 模型 | 用于 | 原因 |
|---|---|---|
| Claude Sonnet 4.6 / GPT-5 | Coordinator、Implementer、Debugger、Reviewer、SecurityAuditor | 需要强推理和编码能力 |
| Claude Haiku 4.5 / Gemini 3 Flash | Planner、Architect、StressTester、TestWriter、QATester | 搜索+分析+测试，轻量够用，成本低 |

---

## 十、反馈循环机制

系统中存在以下反馈循环，由 Coordinator 自动管理：

1. **Plan → Architect → Plan**：Architect 发现问题 → Planner 修订 → Architect 再审
2. **Implement → Review → Implement**：Reviewer < 7.0 → Implementer 修复 → Reviewer 再审（循环直到 ≥ 7.0）
3. **Implement → StressTest → Implement → Review**：压力测试发现缺陷 → 加固 → 重新审查
4. **Implement → SecAudit → Implement → Review + SecAudit**：安全漏洞 → 修复 → 双重再审
5. **Implement → Test → Debug → Implement**：测试失败 → 诊断 → 修复 → 重测
6. **Implement → QA → Implement → QA**：Blocker → 修复 → 重新 QA

---

## 十一、安装方式

### 项目级安装
```bash
mkdir -p .github/agents
cp agents/*.agent.md .github/agents/
mkdir -p docs/project/sprints
```

### 用户全局安装
```bash
# Linux/macOS
mkdir -p ~/.vscode/agents
cp agents/*.agent.md ~/.vscode/agents/

# Windows (PowerShell)
mkdir %USERPROFILE%\.vscode\agents
copy agents\*.agent.md %USERPROFILE%\.vscode\agents\
```

安装后重启 VS Code（`Ctrl+Shift+P` → `Developer: Reload Window`），在 Copilot Chat 输入 `@DevCoordinator` 即可使用。

---

## 十二、Agent 间依赖关系

```
DevCoordinator
  ├── Planner ──────────────────────┐
  │   └── (输出任务列表)              │
  ├── Architect ────────────────────┤
  │   └── (审查 Planner 输出，可并行) │
  ├── Implementer ──────────────────┤
  │   ├── (依赖 Planner 输出)        │
  │   ├── (依赖 Architect 审查结果)   │
  │   └── (错误时触发 Debugger)      │
  ├── Debugger ────────────────────┤
  │   └── (依赖 Implementer 的 #NEEDS_DEBUG 标记)
  ├── Reviewer ────────────────────┤
  │   └── (依赖 Implementer 输出)    │
  ├── StressTester ────────────────┤
  │   └── (依赖 Implementer 输出，可与 SecurityAuditor 并行)
  ├── SecurityAuditor ─────────────┤
  │   └── (依赖 Implementer 输出，可与 StressTester 并行)
  ├── TestWriter ──────────────────┤
  │   └── (依赖 Implementer 输出，可与 QATester 并行)
  └── QATester ────────────────────┘
      └── (依赖 TestWriter 完成)
```

---

## 十三、设计参考来源完整列表

| # | 来源 | URL | 被吸收的设计 |
|---|---|---|---|
| 1 | GitHub Blog: 2500+ repos 分析 | https://github.blog/ai-and-ml/github-copilot/how-to-write-a-great-agents-md-lessons-from-over-2500-repositories/ | 六项核心要素、三层边界(✅⚠️🚫)、代码示例优于文字、命令放前面、具体技术栈 |
| 2 | VS Code Subagents 官方文档 | https://code.visualstudio.com/docs/agents/subagents | Coordinator-Worker 模式、多模型共识、嵌套子代理、runSubagent 工具 |
| 3 | VS Code Custom Agents 官方文档 | https://code.visualstudio.com/docs/agent-customization/custom-agents | .agent.md frontmatter 格式、user-invocable、disable-model-invocation |
| 4 | awesome-copilot Blueprint Mode | https://github.com/github/awesome-copilot/blob/main/agents/blueprint-mode.agent.md | 置信度机制（Confidence Score 0-100）、6 维 Rubric 自检、工具使用纪律、自主性分级 |
| 5 | awesome-copilot ai-team-producer (Remy) | https://github.com/github/awesome-copilot/blob/main/agents/ai-team-producer.agent.md | PROJECT_BRIEF + sprint 文档体系、Bug Triage、进度管理 |
| 6 | awesome-copilot ai-team-qa (Ivy) | https://github.com/github/awesome-copilot/blob/main/agents/ai-team-qa.agent.md | QATester 测试清单、Bug Report 格式（复现步骤/严重程度/影响范围）、QA 签收流程 |
| 7 | awesome-copilot Devils-Advocate | https://github.com/github/awesome-copilot/blob/main/agents/devils-advocate.agent.md | StressTester 攻击维度设计（挑战假设、找缺陷、压测边界） |
| 8 | awesome-copilot Debug Mode | https://github.com/github/awesome-copilot/blob/main/agents/debug.agent.md | Debugger 独立诊断流程（复现→根因→修复判定） |
| 9 | awesome-copilot WG Code Sentinel | https://github.com/github/awesome-copilot/blob/main/agents/wg-code-sentinel.agent.md | SecurityAuditor CWE 编号映射、安全审查输出格式 |
| 10 | GitHub Discussion #192232 | https://github.com/orgs/community/discussions/192232 | AGENTS.md + MEMORY.md 共享上下文、parallelize 并行提示、Handoff 交接 |
| 11 | GitHub Discussion #182197 | https://github.com/orgs/community/discussions/182197 | AI 代码生产 8 条铁律、Human-in-the-Loop、CI 比对 AI 更严、安全独立审计、Agent 约束优于自由 |
| 12 | Zenn: Orchestrator 模式实战 | https://zenn.dev/openjny/articles/e11450f61d067f | 工具最小化原则（Context Rot / Tool Overload）、职责分离表、#tool:todo 引导执行 |
| 13 | DevActivity: AI Agent Orchestration | https://devactivity.com/insights/streamlining-ai-agent-orchestration-in-copilot-chat-a-software-developer-overview | Persistent State（ai-state.json）、Capability Manifests、跨 team 编排 |
| 14 | Medium: Subagent Review Loop | https://medium.com/@xorets/using-github-copilot-subagents-for-review-and-validation-f2b5c41d8987 | specialist+reviewer 反馈循环、Meta-Review 机制、不同模型消除盲点 |

---

## 十四、版本演进历史

| 版本 | 日期 | Agent 数量 | 主要变化 |
|---|---|---|---|
| v1 | 2026-06-10 | 7 | 初版：Coordinator, Planner, Architect, Implementer, Reviewer, SecurityAuditor, TestWriter |
| v2 | 2026-06-10 | 7 | 基于社区最佳实践全面重写 prompt：工具最小化、三层边界、OWASP Top 10 覆盖、输出格式标准化 |
| v3 | 2026-06-10 | 10 | +Debugger, +StressTester, +QATester；Reviewer 升级为 6 维打分制；Coordinator 加入置信度机制 + 项目记忆体系；SecurityAuditor 加入 CWE 映射；Planner 输出加入依赖标注 |
