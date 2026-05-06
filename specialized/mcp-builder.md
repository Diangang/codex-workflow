# MCP 构建

专属 skill。用于构建或修改 MCP 服务、工具、资源、协议集成或外部能力连接。

## 默认叠加

- `common/operating.md`
- `common/verification.md`
- 多步骤实现时叠加 `common/planning.md`

## 深度参考

- `external/raw/superpowers-zh/skills/mcp-builder/SKILL.md`

## 规则

- 先明确 MCP 服务器提供哪些工具、资源或模板。
- 明确输入 schema、输出 schema、错误语义和权限边界。
- 工具应可独立验证。
- 不把未请求的外部服务、认证或状态管理加入实现。

## 输出

包含能力清单、接口约定、实现摘要、测试命令和集成说明。
