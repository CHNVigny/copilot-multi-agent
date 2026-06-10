# Copilot Multi-Agent v5

> 10 个专业 AI Agent 组成的开发团队，在 VS Code + GitHub Copilot 中自动编排完整的软件开发流水线。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)

## 概述

这是一个为 **VS Code + GitHub Copilot** 设计的多角色协作编程 Agent 团队。你只需要说需求，`DevCoordinator` 自动编排 6 阶段开发流水线：

```
规划 → 架构审查 → 实现 → (压力测试 + 安全审计) → 测试 → 收尾
```

融合 **Andrej Karpathy 编程 4 原则**：Think Before Coding / Simplicity First / Surgical Changes / Goal-Driven Execution。

## 🏗 Agent 架构

```
                         ┌──────────────────────────┐
                         │     DevCoordinator v5     │
                         │   总指挥 + 可追溯流水线     │
                         │   置信度决策 + Karpathy 原则 │
                         └──────────┬───────────────┘
                                    │
         ┌─────────┬────────┬───────┼───────┬────────┬──────────┬──────────┐
         ▼         ▼        ▼       ▼       ▼        ▼          ▼          ▼
    ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
    │Plan..│ │Archi.│ │Imple.│ │Debug.│ │Revie.│ │Stress│ │Secur.│ │ Test │
    │      │ │      │ │      │ │      │ │6维打分│ │压力  │ │OWASP │ │Writer│
    └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └──────┘
                                                                    │
                                                              ┌──────┘
                                                              ▼
                                                        ┌──────────┐
                                                        │ QATester │
                                                        │ 手动测试  │
                                                        │ Bug报告   │
                                                        └──────────┘
```

## 📊 Agent 团队

| Agent | 职责 | 触发条件 | 终端 | 模型 |
|---|---|---|---|---|
| **DevCoordinator** | 总指挥，6 阶段流水线 + 项目管理 + 文档分目录 | 所有新任务 | ✅ | Sonnet/GPT-5 |
| **Planner** | 需求拆解，Goal-Driven 格式 | 每次任务 | ❌ | Haiku/Flash |
| **Architect** | 架构一致性审查，发现复用 | 每次任务 | ✅ | Haiku/Flash |
| **Implementer** | Karpathy 原则驱动编码 | 每个实现任务 | ✅ | Sonnet/GPT-5 |
| **Debugger** | 诊断编译/运行错误，小修自干 | 出错时 | ✅ | Sonnet/GPT-5 |
| **Reviewer** | 6 维打分 + Karpathy 审查 | 每个实现完成后 | ✅ | Sonnet/GPT-5 |
| **StressTester** | 并发/边界/降级压力测试 | 复杂逻辑时 | ❌ | Haiku/Flash |
| **SecurityAuditor** | OWASP A01-A10 + CWE Top 25 | 用户输入/认证时 | ✅ | Sonnet |
| **TestWriter** | 单元/集成测试 | 功能实现后 | ✅ | Haiku/Flash |
| **QATester** | 手动测试 + Bug Report | 测试完成后 | ✅ | Haiku/Flash |

## 🚀 快速开始

### 安装

```bash
# 在你的项目根目录（任意项目都行）
mkdir -p .github/agents docs/project/{memory,sprints,pipeline}
cp agents/*.agent.md .github/agents/
```

### 全局安装（所有项目可用）

```bash
# Linux/macOS
mkdir -p ~/.vscode/agents && cp agents/*.agent.md ~/.vscode/agents/

# Windows PowerShell
mkdir %USERPROFILE%\.vscode\agents
copy agents\*.agent.md %USERPROFILE%\.vscode\agents\
```

安装后重启 VS Code（`Ctrl+Shift+P` → `Developer: Reload Window`）。

### 使用

在 Copilot Chat 中输入 `@DevCoordinator <你的需求>`，例如：

> @DevCoordinator 加一个用户修改邮箱的功能，需要发验证码到新邮箱确认

Coordinator 自动走完：规划 → 架构审查 → 编码 → 审查 → 安全审计 → 测试。

也可以单独调用任意 Agent：`@Reviewer` 审查代码、`@SecurityAuditor` 审计安全、`@QATester` 验证功能。

## ✨ 核心特性

- **Coordinator-Worker 编排**：主 Agent 自动分派子 Agent 任务
- **Karpathy 4 原则**：Think Before Coding / Simplicity First / Surgical Changes / Goal-Driven Execution
- **6 维加权打分审查**：正确性×3 + 安全×2 + 性能×1.5 + 可维护性×1.5 + 一致性×1 + 健壮性×1
- **可追溯流水线**：每次任务生成日志文件（含子代理调用记录、阶段追踪表）
- **决策置信度**：≥90 自动 / 70-89 标注 / <70 暂停询问
- **反馈循环**：实现→审查→修复→再审，循环直到 ≥ 7 分
- **工具最小化**：每个 Agent 只配必要 tools，防 Context Rot
- **模型分层**：规划/测试用 Haiku/Flash 省钱，实现/审计用 Sonnet/GPT-5 保质量
- **项目记忆**：`docs/project/memory/` 下 PROJECT_BRIEF + ADR + progress 跨 session 持久化
- **8/10 Agent 拥有 terminal**：Implementer 写完即跑编译，Architect 跑目录/依赖分析

## 🔄 完整工作流示例

```
你: @DevCoordinator 加一个用户头像上传功能

DevCoordinator:
  │
  ├─ 📋 Phase 1: 规划
  │   ├── 创建 docs/project/pipeline/2026-06-11-头像上传.md
  │   ├── Planner → 拆解为 4 个任务 + 验证点
  │   └── Architect → 审查通过 ✅
  │
  ├─ 💻 Phase 2: 实现
  │   ├── Implementer → Task 1-4 逐项完成，编译自验
  │   ├── Reviewer → 8.5/10 ✅ (Karpathy 原则检查通过)
  │   └── 所有子代理调用和分数记录到 pipeline 日志
  │
  ├─ 🔒 Phase 3+4: 安全 + 压力（并行）
  │   ├── SecurityAuditor → 🚨 高危：路径遍历 CWE-22
  │   ├── StressTester → 并发上传文件覆盖风险
  │   ├── Implementer → 路径白名单 + 文件锁
  │   └── 双重再审通过 ✅
  │
  ├─ ✅ Phase 5: 测试
  │   ├── TestWriter → 15 条测试全部通过
  │   └── QATester → 23 项验证，1 Minor Bug → 修复 → 🟢 签收
  │
  └─ 📝 Phase 6: 收尾
      ├── 更新 memory/progress.md
      ├── pipeline 日志标记完成
      └── 向用户汇报完整追踪表
```

## 📁 项目文件结构

```
copilot-multi-agent/
├── agents/                    # 10 个 Agent 配置文件
│   ├── coordinator.agent.md   # 总指挥
│   ├── planner.agent.md       # 任务规划
│   ├── architect.agent.md     # 架构审查
│   ├── implementer.agent.md   # 代码实现 (Karpathy)
│   ├── debugger.agent.md      # 诊断调试
│   ├── reviewer.agent.md      # 6 维打分审查 (Karpathy)
│   ├── stress-tester.agent.md # 压力测试
│   ├── security-auditor.agent.md # OWASP+CWE 安全审计
│   ├── test-writer.agent.md   # 自动测试
│   └── qa-tester.agent.md     # QA 手动测试
├── DJANGO-PLATFORM-DEV.md     # Tenon 平台开发文档 (co-branded)
├── COPILOT-AGENTS-SUMMARY.md  # Agent 团队完整参考文档
├── CHANGELOG.md               # 版本日志
├── CONTRIBUTING.md            # 贡献指南
├── LICENSE                    # MIT
└── README.md                  # 本文件
```

## 📚 设计参考

- GitHub Blog: [How to write a great agents.md](https://github.blog/ai-and-ml/github-copilot/how-to-write-a-great-agents-md-lessons-from-over-2500-repositories/) — 2500+ repos 分析
- awesome-copilot: [Blueprint Mode](https://github.com/github/awesome-copilot/blob/main/agents/blueprint-mode.agent.md), [ai-team-producer](https://github.com/github/awesome-copilot/blob/main/agents/ai-team-producer.agent.md), [ai-team-qa](https://github.com/github/awesome-copilot/blob/main/agents/ai-team-qa.agent.md), [Devils-Advocate](https://github.com/github/awesome-copilot/blob/main/agents/devils-advocate.agent.md), [Debug Mode](https://github.com/github/awesome-copilot/blob/main/agents/debug.agent.md), [WG Code Sentinel](https://github.com/github/awesome-copilot/blob/main/agents/wg-code-sentinel.agent.md)
- [Zenn: Orchestrator Pattern with Sub-Agents](https://zenn.dev/openjny/articles/e11450f61d067f) — 工具最小化、职责分离
- [Medium: Subagent Review Loop](https://medium.com/@xorets/using-github-copilot-subagents-for-review-and-validation-f2b5c41d8987) — specialist+reviewer 反馈循环
- [Andrej Karpathy's LLM Coding Pitfalls](https://github.com/multica-ai/andrej-karpathy-skills) — 4 条编程原则
- GitHub Discussion [#182197](https://github.com/orgs/community/discussions/182197) — AI 代码生产最佳实践

## 版本

当前版本：**v5**。详见 [CHANGELOG.md](./CHANGELOG.md)。

## License

MIT
