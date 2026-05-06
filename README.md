# Codex Workflow

这是一个个人 Codex 工作流技能，用来统一管理多个工程技能的触发、路由和验证规则。

## 用途

- 先通过 `router.md` 判断当前任务应该使用哪个专属 skill 和通用 skill。
- 再通过 `priority.md` 处理用户指令、专属规则、通用规则和外部资料之间的优先级。
- 将外部 skill 保留在 `external/raw/` 作为原始参考资料，默认不直接堆叠进运行入口。

## 目录

- `SKILL.md`：Codex 技能入口。
- `router.md`：任务到 skill 的路由表。
- `priority.md`：规则冲突时的优先级说明。
- `common/`：通用执行、验证、调试、审查、文本自然化等纪律。
- `specialized/`：面向具体领域的专属规则。
- `external/`：外部 skill 来源、吸收状态和原始资料。
- `agents/`：与该工作流相关的 agent 说明。

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

在 Codex 中安装或加载本目录作为 skill 后，遇到开发、调试、review、文本润色、kernel storage、Linux 对齐等任务时，先使用 `workflow` 入口，再按 `router.md` 选择更具体的规则。
