# Kernel Storage 审查

专属 skill。用于 storage/I/O patch、backport patch、性能 patch、bugfix patch 或 Lite/Linux 对齐相关 kernel review。

## 默认叠加

- `common/review.md`
- `common/verification.md`
- 中文输出时叠加 `common/chinese-style.md`

## 深度参考

- `external/raw/kernel-storage-workflow/references/patch-review.md`
- 涉及 Lite/Linux 对齐时读 `specialized/linux-alignment.md`

## 审查顺序

1. 正确性和行为回归。
2. 上下文：process/atomic/IRQ/softirq/RCU、睡眠规则、GFP/reclaim。
3. 并发、锁、生命周期、引用计数、内存序。
4. 错误路径、清理、回滚、hotplug/remove、completion 语义。
5. ABI、sysfs/procfs/uapi、tracepoint、on-disk/wire format。
6. 数据完整性和顺序：flush、FUA、barrier、journaling、crash consistency。
7. 测试、reproducer、debug config 和缺失验证。
8. commit message、patch 拆分、Fixes/stable/backport 说明。

## 输出

问题在前，按严重程度排序。每条问题包含触发条件、风险和建议修复或开放问题。
