# Kernel 路由

| 任务 | 专属 skill | 通用 skill | 需要时深读 |
| --- | --- | --- | --- |
| Lite/Linux 对齐、改 ABI/API/生命周期 | `specialized/linux-alignment.md` | `common/operating.md`, `common/verification.md` | `external/raw/linux-alignment/SKILL.md` |
| Kernel storage 开发 | `specialized/kernel-storage-develop.md` | `common/operating.md`, `common/verification.md`, 可行时 `common/tdd.md` | `external/raw/kernel-storage-workflow/references/development.md`, `external/raw/linux-storage/SKILL.md` |
| Kernel storage 调试 | `specialized/kernel-storage-debug.md` | `common/debugging.md`, `common/verification.md` | `external/raw/kernel-storage-workflow/references/problem-debugging.md`, `external/raw/linux-storage/SKILL.md` |
| Kernel storage review | `specialized/kernel-storage-review.md` | `common/review.md`, `common/verification.md`, 中文场景加 `common/chinese-style.md` | `external/raw/kernel-storage-workflow/references/patch-review.md` |
| Kernel storage backport | `specialized/kernel-storage-backport.md` | `common/operating.md`, `common/verification.md` | `external/raw/kernel-storage-workflow/references/backport.md` |
| Kernel storage 测试 | `specialized/kernel-storage-test.md` | `common/verification.md` | `external/raw/kernel-storage-workflow/references/testing.md` |
| Kernel storage 性能 | `specialized/kernel-storage-performance.md` | `common/operating.md`, `common/verification.md` | `external/raw/kernel-storage-workflow/references/performance-optimization.md` |

## 冲突裁决

- Lite/Linux 对齐优先于 kernel storage 通用规则。
- 开发、backport、测试、性能互相重叠时，按用户目标选择主专属 skill，再叠加其他文件作为参考。
