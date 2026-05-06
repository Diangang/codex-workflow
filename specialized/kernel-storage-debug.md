# Kernel Storage 调试

专属 skill。用于 I/O 慢、I/O error、hang、panic、Oops、死锁、数据损坏、writeback/page-cache 异常、blk-mq stall、NVMe/SCSI/DM 问题。

## 默认叠加

- `common/debugging.md`
- `common/verification.md`

## 深度参考

- `external/raw/kernel-storage-workflow/references/problem-debugging.md`
- 根因复杂时可读 `external/raw/superpowers-zh/skills/systematic-debugging/SKILL.md`
- 需要紧凑 storage 清单时读 `external/raw/linux-storage/SKILL.md`

## 入口数据

- kernel 版本、分支、upstream base、相关 commit。
- 相关 `.config`。
- dmesg、journal、vmcore、trace、perf、blktrace、sysfs/debugfs 快照。
- 硬件与存储拓扑。
- workload/reproducer、复现频率、触发时长、影响范围。

## 流程

1. 先保现场，不先重启或破坏证据。
2. 定位层级：syscall、VFS、FS、page cache/writeback、block、driver、device。
3. 提出 2-4 个可证伪假设。
4. 每个假设给下一步观测和预期结果。
5. 只有理解失败路径和状态变化后才给修复。

## 输出

说明代码路径、关键数据结构或状态、外部症状、证据和验证方式。
