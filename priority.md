# 优先级

本文用于裁决 workflow 内部规则、通用规则、专属规则和外部原始资料之间的冲突。

## 总优先级

1. 用户明确指令、当前仓库指令、系统/开发者指令。
2. 安全、数据完整性、ABI/API 兼容、持久化状态正确性。
3. 专属 skill。
4. 通用 skill。
5. `external/raw/` 下的外部原始资料。

外部原始资料是输入材料。只有当本 workflow 没覆盖细节、需要模板、需要命令清单、或需要核对原文时才深读。

## 多个专属 skill 冲突

按更具体、风险更高者优先：

1. [specialized/linux-alignment.md](specialized/linux-alignment.md)
2. `specialized/kernel-storage-*.md`
3. [specialized/mcp-builder.md](specialized/mcp-builder.md) 或 [specialized/non-kernel-performance.md](specialized/non-kernel-performance.md)
4. 无专属，仅使用通用 skill

示例：如果一个 kernel storage 改动同时涉及 Lite/Linux 对齐，先按 `linux-alignment` 做 reference-first 和 mapping ledger，再叠加 kernel storage 的子系统语义。

## 常见冲突裁决

- 专属流程和通用流程冲突时，专属流程优先；通用流程只补执行纪律。
- 中文表达规则只影响写法，不降低问题严重性，不改变技术结论。
- TDD 是优先选择，不是硬性形式主义；kernel、硬件、故障注入和环境受限任务可以改用明确的源码证据、reproducer、构建、fstests、blktests、fio、日志或追踪验证。
- 未实际运行的测试不能写成“已通过”；只能写“未运行，原因是 ...”或“建议运行 ...”。
- 调试任务必须先证明根因或缩小到可验证假设，再改代码。
- review 任务以问题为先，不为了完整性罗列无关建议。
- subagent 只有在用户明确要求代理、并行代理或委派时才使用；不能因为外部资料建议并行就自动启用。

## 新 skill 纳入规则

新增外部 skill 时，不直接堆进运行入口。先放入 `external/raw/<source>/`，再和现有专属/通用规则对比：

- 已覆盖且无新信息：只在 [external/catalog.md](external/catalog.md) 记录来源。
- 有可复用方法：融合到对应 `common/` 或 `specialized/` 文件。
- 有领域专用价值：新增或更新 `specialized/` 文件，并在 [router.md](router.md) 注册。
- 与现有规则冲突：按本文裁决，必要时在对应文件写清楚选择理由。
