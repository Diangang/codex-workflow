---
name: workflow
description: 当需要在多个技能之间协调个人 Codex 工作、选择工作流、维持稳定工作模式，或处理 Linux kernel 存储、调试、审查、backport、测试等可能涉及多个技能的任务时使用。
---

# 工作流

这是个人 Codex 工作的总入口技能。它不把外部技能并列堆叠起来，而是先把任务路由成“专属 skill + 通用 skill”的组合；冲突时专属 skill 优先。后续所有 skill 相关的新增、修改、吸收和路由调整，也默认在本 workflow 工程中集中管理。

## 核心规则

1. 先读 [router.md](router.md)，再按一级路由读取 `routes/` 下的细分路由，确定专属 skill 和通用 skill。
2. 再按 [priority.md](priority.md) 裁决冲突。
3. 有专属 skill 时先执行专属规则，再叠加通用执行纪律。
4. 无专属 skill 时，只使用匹配的通用 skill。
5. 外部原始 skill 只作为参考资料，默认不直接作为运行入口。
6. 新增或修改 skill 时，优先修改本 workflow 的 `common/`、`specialized/`、`routes/`、`router.md`、`priority.md` 或 `external/catalog.md`；不要新建顶层独立 skill，除非用户明确要求或属于系统内置能力。

## 优先级

1. 用户指令和仓库指令。
2. 安全、数据完整性、ABI/API 兼容、持久化状态正确性。
3. 本工作流的专属 skill。
4. 本工作流的通用 skill。
5. `external/raw/` 下的外部原始资料。

详细冲突规则见 [priority.md](priority.md)。

## 路由

所有实质性任务先读 [router.md](router.md)。

- 专属 skill 放在 `specialized/`。
- 通用 skill 放在 `common/`。
- 长模板和已吸收规则的参考材料放在 `references/`。
- 外部原始资料放在 `external/raw/`，由 workflow 统一管理。
- 外部来源和吸收状态见 [external/catalog.md](external/catalog.md)。
- skill 工程自身的维护规则也属于本 workflow 的一部分；新增能力先作为外部来源记录，再融合到 `common/` 或 `specialized/`。

## 执行契约

对每个实质性任务：

- 说明使用的专属 skill 和通用 skill；没有专属 skill 时说明只使用通用 skill。
- 只收集当前模式需要的上下文。
- 改动范围严格对应用户请求。
- 在宣称完成前，用命令、测试、日志或明确证据验证。
- 缺失数据要标记为待补充，不要猜测补齐。

## 输出契约

默认保持简洁的工程输出：

- 使用了什么模式。
- 做了什么改动，或发现了什么问题。
- 证据或验证结果。
- 剩余风险或待补充数据。
