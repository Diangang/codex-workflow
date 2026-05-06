# Kernel Storage 性能

专属 skill。用于 storage/I/O 吞吐、p99/p99.9 延迟、锁竞争、CPU 开销、队列调度、writeback、blk-mq、NVMe/SCSI 或文件系统性能分析。

## 默认叠加

- `common/operating.md`
- `common/verification.md`

## 深度参考

- `external/raw/kernel-storage-workflow/references/performance-optimization.md`
- 需要测试矩阵时读 `specialized/kernel-storage-test.md`
- 需要改代码时读 `specialized/kernel-storage-develop.md`

## 入口数据

- kernel 版本、分支、config、debug 选项。
- 硬件、NUMA、设备、固件、queue depth、scheduler。
- workload、fio 参数、业务指标、复现时长。
- baseline：IOPS、BW、p50/p99/p99.9、CPU、锁等待、上下文切换。
- trace/perf/blktrace/btt/ftrace/eBPF 数据。

## 流程

1. 明确指标和成功标准。
2. 固定硬件、内核、config、workload，建立 baseline。
3. 先宏观定位 CPU、I/O、内存、调度、锁或设备瓶颈。
4. 再做微观分析。
5. 每次只改一个主要变量。
6. 用多轮 A/B 和完整百分位分布验证。
