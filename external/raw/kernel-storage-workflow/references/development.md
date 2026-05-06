# Development Reference

本文件用于编写、修改或设计 Linux 内核存储与 I/O 子系统 patch。目标是让代码变更从一开始就带着数据路径、上下文、生命周期和验证计划。

## Primary References

* VFS：`https://docs.kernel.org/filesystems/vfs.html`
* blk-mq：`https://docs.kernel.org/block/blk-mq.html`
* Queue sysfs：`https://docs.kernel.org/block/queue-sysfs.html`
* Memory allocation：`https://docs.kernel.org/core-api/memory-allocation.html`
* GFP masks from FS/IO context：`https://docs.kernel.org/core-api/gfp_mask-from-fs-io.html`
* Submitting patches：`https://docs.kernel.org/process/submitting-patches.html`
* Submit checklist：`https://docs.kernel.org/process/submit-checklist.html`
* Coding style：`https://docs.kernel.org/process/coding-style.html`

## 1. 开发入口检查

动代码前必须回答：

1. 变更所在层级：VFS、FS、Page Cache/Writeback、Block、Driver、Hardware。
2. 入口上下文：process context、atomic、IRQ、softirq、workqueue、RCU read-side。
3. 可睡眠性：是否持有 spinlock、禁抢占、关中断、持有 filesystem transaction 或 reclaim 相关锁。
4. 对象生命周期：谁分配、谁发布、谁持有引用、谁释放、是否跨 CPU/线程/work/timer/completion。
5. 错误路径：每个失败点如何释放资源、回滚状态、向上返回哪个 errno 或 `blk_status_t`。
6. 用户可见面：uapi/ioctl/sysfs/procfs/debugfs/tracepoint/on-disk format 是否受影响。
7. 验证方式：最小 reproducer、KUnit/kselftest/fstests/blktests/fio/fault injection 哪些必跑。

若无法回答上下文或生命周期，不要先写大 patch；先补源码阅读、trace 或最小实验。

## 2. 数据路径骨架

常见写路径：

```text
write()/pwritev()/io_uring
  -> VFS file_operations::write_iter
  -> filesystem buffered/direct I/O
  -> Page Cache / iomap / writeback
  -> submit_bio()
  -> blk_mq_submit_bio()
  -> request merge/plug/scheduler
  -> blk_mq_hw_ctx dispatch
  -> driver queue_rq()
  -> device completion
  -> bio/request endio
```

常见 fsync 路径：

```text
fsync()/fdatasync()
  -> file_operations::fsync
  -> filesystem journal/log commit or metadata flush
  -> blk layer REQ_PREFLUSH/FUA if required
  -> completion/error propagation
```

开发时要把 patch 对应到具体路径，例如 `fs/ext4/file.c:ext4_file_write_iter()`、`fs/iomap/buffered-io.c`、`mm/filemap.c`、`block/blk-mq.c`、`drivers/nvme/host/pci.c`。

## 3. 分层开发规则

### 3.1 VFS / Filesystem

* 明确改的是 VFS contract 还是某个 filesystem 的实现；不要把 ext4/XFS/btrfs 的局部语义泛化为所有 FS。
* 检查 `struct inode`、`struct file`、`struct address_space`、`struct super_block` 的引用和锁保护。
* Buffered I/O 要考虑 folio/page lock、dirty 标记、writeback、truncate/invalidate、mmap 并发。
* Direct I/O 要考虑 page pin、alignment、iomap/bio submit、与 buffered I/O 的一致性边界。
* `fsync`/`fdatasync` 修改必须说明 metadata、data、journal/log、flush/FUA 的持久化语义。
* on-disk format 或 feature flag 变更必须说明旧内核/新内核读写兼容性、fsck 工具需求和升级路径。

### 3.2 Page Cache / Writeback / MM

* 新增分配点先判断是否允许睡眠；`GFP_KERNEL` 可触发 direct reclaim，调用上下文必须允许睡眠。
* FS/IO 递归死锁风险不要用 `GFP_NOFS`/`GFP_NOIO` "just in case" 掩盖；优先用 `memalloc_nofs_save()` 或 `memalloc_noio_save()` 包住真实 critical section，并写清原因。
* 在 reclaim、writeback、journal transaction、block submit 路径中新增等待点，要检查是否可能等待自己持有的资源。
* folio/page 引用、lock/unlock、dirty/writeback 状态转换必须在错误路径中成对处理。
* 对 memcg/PSI/dirty throttling 有影响的变更，要在测试计划中加入压力与尾延迟观测。

### 3.3 Block Layer / blk-mq

* 区分 `struct bio` 和 `struct request`：一个 request 可包含一个或多个 bio，bio 提交后可能被合并、plug 或调度。
* blk-mq 有 software staging queue (`struct blk_mq_ctx`) 和 hardware dispatch queue (`struct blk_mq_hw_ctx`)；性能或 hang 分析要说明卡在 ctx、scheduler、hctx dispatch、driver queue 还是 completion。
* 不假设 completion 顺序等于 submission 顺序；需要顺序语义时必须由上层机制保证。
* 修改 `queue_rq()`、budget、tag、timeout、poll、completion、quiesce/freeze 路径时，检查失败返回后谁负责重新入队、释放 budget、完成 request。
* 修改 queue limits 或 scheduler/sysfs 行为时，确认对 stacked device、partition、blk-cgroup、zoned device 的影响。
* stacking driver 克隆 bio/request 时，确认 clone 与 original 的完成顺序、page 引用和 error propagation。

### 3.4 NVMe / SCSI / Driver

* 区分 host timeout、controller reset、transport error、media error、firmware behavior。
* 队列深度、tag、IRQ affinity、polling、multipath 相关变更必须说明 CPU/NUMA 和硬件队列映射。
* timeout/reset/error recovery 路径要验证 inflight request 如何完成、重试、失败或取消。
* DMA buffer、PRP/SGL、alignment、device limits 相关变更要检查 bounce/swiotlb、IOMMU 和 queue limits。

## 4. 上下文和生命周期检查清单

### 4.1 睡眠与锁

| 问题 | 必查点 |
|---|---|
| 可能睡眠 | `might_sleep()`、mutex、wait、memory allocation、copy_to/from_user |
| atomic 上下文 | spinlock、preempt disabled、irq disabled、softirq、RCU read-side |
| 锁序 | 新锁是否和已有锁形成 ABBA，是否需要 lockdep class |
| 回调生命周期 | work/timer/completion/RCU callback 是否能访问已释放对象 |

### 4.2 引用计数

每个对象画出：

```text
allocate/get -> initialize -> publish -> concurrent use -> unpublish -> drain callbacks -> put/free
```

重点检查：

* 错误路径是否 put/free 完整
* publish 前对象是否完全初始化
* 跨线程传递前是否额外 get
* RCU 读侧是否配合 `rcu_dereference()` / `rcu_assign_pointer()`
* teardown 是否先阻止新请求，再等待 inflight，再释放对象

### 4.3 错误码和状态

* syscall 层返回 `-errno`；block 层常用 `blk_status_t`；驱动层需要正确转换。
* 错误路径不能只打印日志；必须把失败向上层传播或完成 request/bio。
* cleanup label 顺序应与资源获取顺序相反。
* partial success 要说明哪些状态已经对外可见，回滚是否可能。

## 5. 变更类型到验证

| 变更类型 | 最低验证 |
|---|---|
| VFS/FS buffered I/O | reproducer + fstests generic + 目标 FS 相关组 + dmesg clean |
| Direct I/O / iomap | alignment 矩阵 + buffered/direct 混合 + fstests dio 相关用例 |
| fsync / journal / log | crash consistency 测试 + dm-log-writes 或 shutdown 类测试 |
| block layer | blktests block + 相关 fio + fault injection |
| NVMe/SCSI driver | blktests nvme/scsi + timeout/reset/error log 验证 |
| 锁/生命周期 | KASAN/KCSAN/lockdep/debug config + 并发压力 |
| 性能优化 | A/B 多轮 fio/perf/trace + 功能回归无新增失败 |

## 6. Patch 输出契约

交付 patch 或设计时包含：

1. 变更目标和 non-goals。
2. 旧路径与新路径的调用链差异。
3. 关键结构体、引用计数、锁和上下文说明。
4. 错误路径和资源释放说明。
5. 用户可见接口或兼容性影响。
6. 测试证据：命令、配置、结果、日志路径。
7. 风险、回滚方案和未覆盖项。
