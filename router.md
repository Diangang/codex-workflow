# 路由

本文是 `workflow` 的入口路由。先选一个最具体、风险最高的主路由，再按需要叠加通用规则；冲突按 [priority.md](priority.md) 裁决。

## 基本规则

- 有专属 skill 时，必须先读专属 skill。
- 通用 skill 只补充执行纪律、验证、审查、调试方法和中文表达。
- 无专属 skill 时，只使用匹配的通用 skill。
- 外部原始资料只作为深读参考，不作为默认入口。
- 任务跨多个领域时，先选最具体、风险最高的专属 skill。

## 一级路由

| 任务类型 | 读取文件 |
| --- | --- |
| Kernel、Lite/Linux 对齐、storage/I/O、SCSI/NVMe/block/filesystem | [routes/kernel.md](routes/kernel.md) |
| 通用开发、调试、review、文本润色、测试补充、收尾 | [routes/general.md](routes/general.md) |
| ByteDance SSH、Prompt 工程、MCP、skill 工程维护、非内核性能 | [routes/tools.md](routes/tools.md) |

## 固定叠加

- 任何完成、修复、测试通过或审查结论：叠加 `common/verification.md`。
- 中文输出、中文文档或中文审查：叠加 `common/chinese-style.md`。
- 文档、说明、PR 描述、cover letter、报告或对外文案：叠加 `common/humanizer.md`。
- 多阶段、多文件、多验证面的任务：叠加 `common/planning.md`。

## Skill 工程维护

- 后续所有 skill 相关修改都集中在本 workflow 工程管理。
- 新能力默认先放入 `external/raw/<source>/` 或在 [external/catalog.md](external/catalog.md) 登记来源，再吸收到 `common/` 或 `specialized/`。
- 不新增顶层独立 skill 入口，除非用户明确要求独立安装，或该 skill 属于 `.system/` 内置能力。
- 修改路由、优先级、通用规则或专属规则后，同步更新 [external/catalog.md](external/catalog.md) 和必要的 README 说明。

## 输出要求

开始实质性任务时，简短说明本轮使用的专属 skill 和通用 skill。没有专属 skill 时，说明只使用通用 skill。
