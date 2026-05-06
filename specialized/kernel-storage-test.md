# Kernel Storage 测试

专属 skill。用于 storage/I/O 测试计划、测试报告、回归验证、压力测试、故障注入、fstests/blktests/fio 或 kernel patch 验证。

## 默认叠加

- `common/verification.md`

## 深度参考

- `external/raw/kernel-storage-workflow/references/testing.md`
- 性能相关时读 `specialized/kernel-storage-performance.md`

## 测试意图

- 正确性和回归。
- 错误路径和故障注入。
- 并发和竞态。
- 稳定性和 soak。
- 性能、尾延迟或 CPU 开销。
- 兼容性或 ABI 行为。

## 测试矩阵

- kernel 版本、分支、config、debug 选项。
- 文件系统、挂载参数、block scheduler、设备类型。
- 硬件拓扑、queue depth、固件、控制器/驱动。
- workload：fio、fsstress、fstests、blktests、KUnit、kselftest、自定义 reproducer。

## 成功标准

无 WARN/BUG/Oops/panic；无 KASAN/KCSAN/lockdep；测试通过计数明确；相关场景 fsck clean 且数据校验通过。
