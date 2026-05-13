# Skill 工程维护

专属 skill。用于新增、修改、吸收、删除、迁移或整理 skill，以及调整 workflow 的路由、优先级、通用规则和专属规则。

## 默认叠加

- `common/operating.md`
- `common/verification.md`

## 原则

- 后续所有 skill 相关修改都集中在本 workflow 工程管理。
- 不新增顶层独立 skill 入口，除非用户明确要求独立安装，或该 skill 属于 `.system/` 内置能力。
- 新能力先登记到 `external/catalog.md`，必要时保留原始资料到 `external/raw/`。
- 可复用规则吸收到 `common/` 或 `specialized/`。
- 触发入口统一写入 `router.md` 或 `routes/` 下的细分路由。

## 流程

1. 判断是外部 skill 吸收、现有规则修改、路由调整，还是结构整理。
2. 对比现有 `common/` 和 `specialized/`，避免重复规则。
3. 只吸收新增且有价值的规则；外部原文只作为深读资料。
4. 更新对应路由和 `external/catalog.md`。
5. 如果影响用户可见工作流，同步更新 `README.md` 或 `SKILL.md`。

## 输出

说明改了哪些 workflow 文件、移除了哪些顶层入口、保留了哪些外部资料，以及剩余风险。
