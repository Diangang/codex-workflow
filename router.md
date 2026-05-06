# 路由

本文是 `workflow` 的入口路由。每个任务先选专属 skill，再叠加通用 skill；如果规则冲突，按 [priority.md](priority.md) 裁决。

## 基本规则

- 有专属 skill 时，必须先读专属 skill。
- 通用 skill 只补充执行纪律、验证、审查、调试方法和中文表达。
- 无专属 skill 时，只使用匹配的通用 skill。
- 外部原始资料只作为深读参考，不作为默认入口。
- 任务跨多个领域时，先选最具体、风险最高的专属 skill。

## 常用路由

| 任务 | 专属 skill | 通用 skill | 需要时深读 |
| --- | --- | --- | --- |
| Lite/Linux 对齐、改 ABI/API/生命周期 | [specialized/linux-alignment.md](specialized/linux-alignment.md) | [common/operating.md](common/operating.md), [common/verification.md](common/verification.md) | `external/raw/linux-alignment/SKILL.md` |
| Kernel storage 开发 | [specialized/kernel-storage-develop.md](specialized/kernel-storage-develop.md) | [common/operating.md](common/operating.md), [common/verification.md](common/verification.md), 可行时 [common/tdd.md](common/tdd.md) | `external/raw/kernel-storage-workflow/references/development.md`, `external/raw/linux-storage/SKILL.md` |
| Kernel storage 调试 | [specialized/kernel-storage-debug.md](specialized/kernel-storage-debug.md) | [common/debugging.md](common/debugging.md), [common/verification.md](common/verification.md) | `external/raw/kernel-storage-workflow/references/problem-debugging.md`, `external/raw/linux-storage/SKILL.md` |
| Kernel storage review | [specialized/kernel-storage-review.md](specialized/kernel-storage-review.md) | [common/review.md](common/review.md), [common/verification.md](common/verification.md), 中文场景加 [common/chinese-style.md](common/chinese-style.md) | `external/raw/kernel-storage-workflow/references/patch-review.md` |
| Kernel storage backport | [specialized/kernel-storage-backport.md](specialized/kernel-storage-backport.md) | [common/operating.md](common/operating.md), [common/verification.md](common/verification.md) | `external/raw/kernel-storage-workflow/references/backport.md` |
| Kernel storage 测试 | [specialized/kernel-storage-test.md](specialized/kernel-storage-test.md) | [common/verification.md](common/verification.md) | `external/raw/kernel-storage-workflow/references/testing.md` |
| Kernel storage 性能 | [specialized/kernel-storage-performance.md](specialized/kernel-storage-performance.md) | [common/operating.md](common/operating.md), [common/verification.md](common/verification.md) | `external/raw/kernel-storage-workflow/references/performance-optimization.md` |
| MCP 构建 | [specialized/mcp-builder.md](specialized/mcp-builder.md) | [common/operating.md](common/operating.md), [common/verification.md](common/verification.md), 多步任务加 [common/planning.md](common/planning.md) | `external/raw/superpowers-zh/skills/mcp-builder/SKILL.md` |
| 非内核性能问题 | [specialized/non-kernel-performance.md](specialized/non-kernel-performance.md) | [common/operating.md](common/operating.md), [common/verification.md](common/verification.md) | 按项目技术栈选择原始资料 |
| 通用开发 | 无 | [common/operating.md](common/operating.md), [common/verification.md](common/verification.md), 可行时 [common/tdd.md](common/tdd.md), 多步任务加 [common/planning.md](common/planning.md) | `external/raw/andrej-karpathy-skills/skills/karpathy-guidelines/SKILL.md`, `external/raw/superpowers-zh/skills/test-driven-development/SKILL.md` |
| 通用调试 | 无 | [common/debugging.md](common/debugging.md), [common/verification.md](common/verification.md) | `external/raw/superpowers-zh/skills/systematic-debugging/SKILL.md` |
| 通用 review | 无 | [common/review.md](common/review.md), [common/verification.md](common/verification.md), 中文场景加 [common/chinese-style.md](common/chinese-style.md) | `external/raw/superpowers-zh/skills/chinese-code-review/SKILL.md` |
| 文本润色、去 AI 味、Humanizer | 无 | [common/humanizer.md](common/humanizer.md), 中文场景加 [common/chinese-style.md](common/chinese-style.md), 涉及事实核验加 [common/verification.md](common/verification.md) | `external/raw/humanizer/SKILL.md` |
| 删除、移除、清理 | 按领域选择；Lite/Linux 对齐必须用 linux-alignment | [common/operating.md](common/operating.md), [common/verification.md](common/verification.md) | 按被删除对象所属领域选择 |
| 测试设计或补测试 | 按领域选择 | [common/verification.md](common/verification.md), 可行时 [common/tdd.md](common/tdd.md) | `external/raw/superpowers-zh/skills/test-driven-development/SKILL.md` |
| 分支收尾、提交前整理 | 无 | [common/finishing.md](common/finishing.md), [common/verification.md](common/verification.md), 中文提交加 [common/chinese-style.md](common/chinese-style.md) | `external/raw/superpowers-zh/skills/finishing-a-development-branch/SKILL.md`, `external/raw/superpowers-zh/skills/chinese-commit-conventions/SKILL.md` |

## 输出要求

开始实质性任务时，简短说明本轮使用的专属 skill 和通用 skill。没有专属 skill 时，说明只使用通用 skill。
