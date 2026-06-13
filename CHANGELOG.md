# Changelog

## v6 (2026-06-13)

### 核心理念升级
基于 Anthropic Building Effective Agents + Teamday Complete Guide + Karpathy 4 原则 + 7 Design Patterns 的系统性改造。
核心目标：**任务稳定性 & 代码可靠性**。

### Added
- **ContextManager Agent (v1)**：为每个子代理生成精准 Context Summary，解决"Agent 失忆"问题
- **循环上限机制**：Implementer 最多 3 轮、其他 Agent 最多 2 轮，防止死循环
- **升级机制 (Escalation)**：重试耗尽后整理"已做尝试+失败原因"向用户精确求助，附带 2-3 个建议方案
- **Phase 2.5 架构后审**：实现完成后 Architect 做二次架构一致性审查
- Pipeline 日志新增「重试计数」和「🆘 升级」状态

### Changed
- **Implementer v4 → v5**：引入 Self-Reflection 循环（编码→自验证→自审查→产出验证→修正→再循环，最多 2 轮）
  - 强制 7 条 Self-Reflection 检查表（不可跳过任何一条）
  - 产出终端输出强制验证：`ls -la` + `test -s` 确认文件真实存在且非空
  - 自己把问题消灭在交付前，不给 Coordinator 留"声称完成但文件缺失"的机会
- **Reviewer v4 → v5**：从"打分+打回去"升级为 Evaluator-Optimizer 模式
  - 行级 before/after 修复指令，附带验证命令
  - Implementer 收到后可以无脑执行
- **Architect v3 → v4**：新增 Phase 2 实现审查模式
  - Phase 1 审查计划 + Phase 2 审查实现，双重验证
- **StressTester v3 → v4**：新增 terminal 权限
  - 实际执行并发压测/资源耗尽/异常输入命令
  - 理论分析 + 实际执行双重覆盖
- **Coordinator v5 → v6**：重构为弹性流水线
  - Context Summary 机制（每次调度子代理前注入状态摘要）
  - 去掉独立的子代理产出验证步骤（信任 Implementer v5 的 Self-Reflection）
  - 精简文档产出验证（Phase 1 + Phase 6 两个检查点）
  - Agents 列表新增 ContextManager

### Agent 权限汇总 (v6)

| Agent | read | search | edit | terminal | agent |
|---|---|---|---|---|---|
| Coordinator | ✅ | ✅ | ✅ | ✅ | ✅ |
| Planner | ✅ | ✅ | ❌ | ❌ | ❌ |
| Architect | ✅ | ✅ | ❌ | ✅ | ❌ |
| ContextManager | ✅ | ✅ | ✅ | ❌ | ❌ |
| Implementer | ✅ | ✅ | ✅ | ✅ | ❌ |
| Debugger | ✅ | ✅ | ✅ | ✅ | ❌ |
| Reviewer | ✅ | ✅ | ❌ | ✅ | ❌ |
| StressTester | ✅ | ✅ | ❌ | ✅ | ❌ |
| SecurityAuditor | ✅ | ✅ | ❌ | ✅ | ❌ |
| TestWriter | ✅ | ✅ | ✅ | ✅ | ❌ |
| QATester | ✅ | ✅ | ❌ | ✅ | ❌ |

总计 11 Agents | 8 个拥有 terminal | 2 个拥有 edit | 1 个拥有 agent
### Added
- 文档分门别类：`docs/project/memory/`、`sprints/`、`pipeline/` 三层目录
- 流水线日志：每次任务生成 `YYYY-MM-DD-任务简称.md`，含子代理调用日志表和阶段追踪表
- Stale 检测：子代理超过 3 轮无返回时主动追踪
- 初始化检查清单：Coordinator 对话开始时检查 5 项目录和文件
- Coordinator 每完成一个 Phase 汇报进度（阶段追踪表格格式）

### Changed
- Coordinator `docs/project/` 全部路径重构为 `docs/project/memory/`
- Architect/Implementer/Reviewer 中 decisions.md 路径同步更新

## v4 (2026-06-10)
### Added
- 融入 Andrej Karpathy 编程 4 原则（Think Before Coding / Simplicity First / Surgical Changes / Goal-Driven Execution）
- Coordinator 新增 Karpathy 4 原则章节作为全局行为准则
- Implementer 重写为 Karpathy 原则驱动版本
- Reviewer 新增 Karpathy 原则审查（4 条检查点，每个 ❌ 即阻塞）
- Planner 任务步骤改为「步骤 → 验证: 检查点」格式

### Changed
- Implementer 精简：去掉了与 Karpathy 原则重复的规则

## v3 (2026-06-10)
### Added
- + Debugger Agent：独立诊断编译/运行时错误
- + StressTester Agent：并发/边界/降级/组合爆炸压力测试
- + QATester Agent：手动测试 + Bug Report
- Reviewer 升级为 6 维度加权打分制（正确性×3 + 安全×2 + 性能×1.5 + 可维护性×1.5 + 一致性×1 + 健壮性×1）
- Coordinator 决策置信度机制（≥90 自动 / 70-89 标注 / <70 询问）
- 项目记忆体系（PROJECT_BRIEF + sprints + decisions + progress）
- SecurityAuditor 额外映射 CWE Top 25
- Planner 输出增加依赖与顺序标注
- Implementer 和 Architect 增加 terminal 权限

## v2 (2026-06-10)
### Changed
- 基于 GitHub Blog (2500+ repos) 分析全面重写 7 个 Agent prompt
- 工具最小化（每个 Agent 只配必要 tools，防 Context Rot）
- 三层边界（✅ ⚠️ 🚫）
- OWASP Top 10 全覆盖
- 输出格式标准化

## v1 (2026-06-10)
### Added
- 初始 7 Agent：Coordinator、Planner、Architect、Implementer、Reviewer、SecurityAuditor、TestWriter
- Coordinator-Worker 编排模式
