# Codex Workflow

这是一个个人 Codex 工作流技能，用来统一管理多个工程技能的触发、路由和验证规则。

## 用途

- 先通过 `router.md` 判断当前任务应该使用哪个专属 skill 和通用 skill。
- 再通过 `priority.md` 处理用户指令、专属规则、通用规则和外部资料之间的优先级。
- 将外部 skill 保留在 `external/raw/` 作为原始参考资料，默认不直接堆叠进运行入口。
- 统一管理后续所有 skill 新增、修改、吸收和路由调整。

## 目录

- `SKILL.md`：Codex 技能入口。
- `router.md`：一级路由入口。
- `routes/`：按领域拆分的细分路由表。
- `priority.md`：规则冲突时的优先级说明。
- `common/`：通用执行、验证、调试、审查、文本自然化等横切纪律。
- `specialized/`：面向具体领域的专属规则。
- `references/`：已吸收规则的长模板或参考材料。
- `external/`：外部 skill 来源、吸收状态和原始资料。
- `agents/`：与该工作流相关的 agent 说明。

## Skill 工程维护

后续所有 skill 相关修改默认集中在本 workflow 工程内完成：

- 新能力先登记到 `external/catalog.md`，必要时保留原始资料到 `external/raw/`。
- 可复用规则吸收到 `common/` 或 `specialized/`。
- 触发入口统一写入 `router.md` 或 `routes/` 下的细分路由。
- 不新增顶层独立 skill，除非用户明确要求独立安装，或属于 `.system/` 内置能力。

## 外部资料

`external/raw/` 下部分目录以 Git submodule 形式保存，GitHub 会显示为可跳转的外部仓库链接：

- `andrej-karpathy-skills`
- `humanizer`
- `superpowers`
- `superpowers-zh`

拉取仓库后如需同步这些外部资料：

```bash
git submodule update --init --recursive
```

## 使用

在 Codex 中安装或加载本目录作为 skill 后，遇到开发、调试、review、内部 SSH 访问、文本润色、Prompt 工程、kernel storage、Linux 对齐等任务时，先使用 `workflow` 入口，再按 `router.md` 选择更具体的规则。
