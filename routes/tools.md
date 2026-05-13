# 工具与内部能力路由

| 任务 | 专属 skill | 通用 skill | 需要时深读 |
| --- | --- | --- | --- |
| ByteDance SSH/root 登录、IPv6 内部机器、jump.byted.org | `specialized/bytedance-ssh-access.md` | `common/verification.md` | 无 |
| Prompt 生成、改写、优化、框架选择 | `specialized/prompt-engineering.md` | 涉及事实核验加 `common/verification.md` | `external/raw/prompt-architect/SKILL.md`, `references/openai-prompt-templates.md` |
| MCP 构建 | `specialized/mcp-builder.md` | `common/operating.md`, `common/verification.md`, 多步任务加 `common/planning.md` | `external/raw/superpowers-zh/skills/mcp-builder/SKILL.md` |
| Skill 新增、修改、吸收、路由调整 | `specialized/skill-maintenance.md` | `common/operating.md`, `common/verification.md` | `.system/skill-creator/SKILL.md`, `external/catalog.md` |
| 非内核性能问题 | `specialized/non-kernel-performance.md` | `common/operating.md`, `common/verification.md` | 按项目技术栈选择原始资料 |

## 冲突裁决

- SSH 访问是工具能力，不与 kernel/storage 任务竞争；涉及远端机器时作为前置能力叠加。
- Prompt 工程只负责产出 prompt，不代表执行 prompt 中的工程任务。
- Skill 工程维护优先保持 workflow 集中管理，不新建顶层 skill。
