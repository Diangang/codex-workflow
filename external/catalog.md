# 外部来源

这里记录外部 skill 的原始来源和吸收状态。`external/raw/` 是资料库，不是默认运行入口。

## 原始目录

- `external/raw/andrej-karpathy-skills/`：Andrej Karpathy guidelines 源码，通用编码纪律已吸收到 `common/operating.md`、`common/verification.md`。
- `external/raw/kernel-storage-workflow/`：kernel storage 专业工作流，已拆分吸收到 `specialized/kernel-storage-*.md`。
- `external/raw/humanizer/`：文本自然化和去 AI 写作痕迹规则，已吸收到 `common/humanizer.md`。
- `external/raw/linux-alignment/`：Lite/Linux 对齐专属规则，已吸收到 `specialized/linux-alignment.md`。
- `external/raw/linux-storage/`：Linux storage 子系统知识，作为 kernel storage 任务的深读资料。
- `external/raw/superpowers-zh/`：中文版本的通用工程方法，已吸收到 `common/`、`specialized/mcp-builder.md` 和中文表达规则。
- `external/raw/superpowers/`：英文原版通用工程方法，主要作为 `superpowers-zh` 的对照资料。

## 当前吸收结果

- 通用执行纪律：`common/operating.md`
- 通用调试：`common/debugging.md`
- 通用验证：`common/verification.md`
- 通用 review：`common/review.md`
- 通用计划：`common/planning.md`
- TDD：`common/tdd.md`
- 中文表达：`common/chinese-style.md`
- 文本自然化 / Humanizer：`common/humanizer.md`
- 收尾：`common/finishing.md`
- Lite/Linux 对齐：`specialized/linux-alignment.md`
- Kernel storage 开发、调试、review、backport、测试、性能：`specialized/kernel-storage-*.md`
- MCP 构建：`specialized/mcp-builder.md`
- 非内核性能：`specialized/non-kernel-performance.md`

## 后续维护

新增 skill 时按以下顺序处理：

1. 放入 `external/raw/<source>/`，保留原貌。
2. 和 `common/`、`specialized/` 现有内容对比。
3. 只把新增且有价值的规则融合进 workflow。
4. 在 [router.md](../router.md) 增加或调整路由。
5. 在本文更新来源和吸收状态。
