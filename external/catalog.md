# 外部来源

`external/raw/` 是资料库，不是默认运行入口。外部 skill 或资料只有在 workflow 没覆盖细节、需要模板、命令清单或原文核对时才深读。

## 原始外部资料

- `external/raw/andrej-karpathy-skills/`：通用编码纪律来源，已吸收到 `common/operating.md`、`common/verification.md`。
- `external/raw/kernel-storage-workflow/`：kernel storage 专业工作流来源，已拆分吸收到 `specialized/kernel-storage-*.md`。
- `external/raw/humanizer/`：文本自然化和去 AI 写作痕迹规则来源，已吸收到 `common/humanizer.md`。
- `external/raw/linux-alignment/`：Lite/Linux 对齐规则来源，已吸收到 `specialized/linux-alignment.md`。
- `external/raw/linux-storage/`：Linux storage 子系统知识，作为 kernel storage 任务的深读资料。
- `external/raw/prompt-architect/`：Prompt Architect 框架资料，作为 `specialized/prompt-engineering.md` 的深读资料。
- `external/raw/superpowers-zh/`：中文通用工程方法来源，已吸收到 `common/`、`specialized/mcp-builder.md` 和中文表达规则。
- `external/raw/superpowers/`：英文原版通用工程方法，主要作为 `superpowers-zh` 的对照资料。

## 已吸收能力

- 通用执行纪律：`common/operating.md`
- 通用验证：`common/verification.md`
- 通用调试：`common/debugging.md`
- 通用 review：`common/review.md`
- 通用计划：`common/planning.md`
- TDD：`common/tdd.md`
- 中文表达：`common/chinese-style.md`
- 文本自然化 / Humanizer：`common/humanizer.md`
- 开发收尾：`common/finishing.md`
- ByteDance SSH 访问：`specialized/bytedance-ssh-access.md`
- Prompt 工程：`specialized/prompt-engineering.md`
- Skill 工程维护：`specialized/skill-maintenance.md`
- OpenAI 风格 Prompt 模板：`references/openai-prompt-templates.md`
- Lite/Linux 对齐：`specialized/linux-alignment.md`
- Kernel storage 开发、调试、review、backport、测试、性能：`specialized/kernel-storage-*.md`
- MCP 构建：`specialized/mcp-builder.md`
- 非内核性能：`specialized/non-kernel-performance.md`

## 顶层入口策略

- `/data00/home/lidiangang/.codex/skills/` 下默认只保留 `.system/` 和 `workflow/`。
- 不新增顶层独立 skill，除非用户明确要求独立安装，或属于 `.system/` 内置能力。
- 新能力先进入本 workflow：登记来源、吸收规则、更新路由。

## 新增或吸收流程

1. 判断是外部 skill 吸收、现有规则修改、路由调整，还是结构整理。
2. 必要时把原始资料放入 `external/raw/<source>/`，保留原貌。
3. 和 `common/`、`specialized/` 现有内容对比，避免重复规则。
4. 只把新增且有价值的规则融合进 workflow 的 `common/` 或 `specialized/`。
5. 在 `router.md` 或 `routes/` 下增加或调整路由。
6. 在本文更新来源和吸收状态。
7. 如果改动影响用户可见工作流，同步更新 `README.md` 或 `SKILL.md`。
