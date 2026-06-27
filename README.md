# Hermes 子Agent编排 🚀

delegate_task 并行执行 + 上下文注入 + 代码清理 + 两阶段审查。

## 功能
- 并行委派：最多3个子Agent同时跑
- 上下文注入：自动检测目录 AGENTS.md 规则
- 代码清理：3 Agent 并行审查
- 两阶段审查：规格合规 → 代码质量
- 失败恢复：自动分析 exit_reason

## 安装
```bash
git clone https://github.com/remix2111/hermagent-subagent-driven-development.git
cp -r hermagent-subagent-driven-development/skills/subagent-driven-development ~/.hermes/skills/
```

## 使用
/skill subagent-driven-development

## License
MIT