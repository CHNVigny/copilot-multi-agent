# Changelog

## v5 (2026-06-11)
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
