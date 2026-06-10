# Contributing

欢迎贡献！你可以：

## 提交 Agent 改进

1. Fork 本仓库
2. 修改 `.agent.md` 文件
3. 提交 PR，说明改了什么、为什么改、参考来源

## 新增 Agent

如果你有新的角色需求（如 DevOps、Documenter、Perf Profiler），可以：

1. 在 `agents/` 下创建 `角色名.agent.md`
2. 在 `coordinator.agent.md` 的 `agents:` 列表中添加新角色名
3. 如果新角色需要独立调用，设 `user-invocable: true`

## 审查标准

所有 PR 应满足：

- [ ] 每个 Agent 的 tools 列表只包含必要工具
- [ ] 每个 Agent 有明确的 name、description、model、触发条件
- [ ] 输出格式有明确规定
- [ ] 不引入与 Karpathy 原则冲突的规则

## 参考

本项目的设计参考了以下来源（详见 `COPILOT-AGENTS-SUMMARY.md` 第十三章）：

- GitHub Blog: How to write a great agents.md (2500+ repos 分析)
- awesome-copilot: Blueprint Mode, ai-team-producer/qa, Devils-Advocate, Debug Mode, WG Code Sentinel
- Zenn: Orchestrator Pattern with Sub-Agents
- Medium: Subagent Review Loop
- Andrej Karpathy's observations on LLM coding pitfalls
