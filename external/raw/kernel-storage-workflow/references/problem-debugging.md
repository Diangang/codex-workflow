# Problem Debugging Reference

本文件为内核存储与 I/O 子系统问题定位提供详细方法论。适用场景：IO 慢、IO error、Oops、Panic、Hang、死锁、数据损坏、内存泄漏、UAF、越界访问。

## Primary References

* Linux kernel patch 提交：`https://docs.kernel.org/process/submitting-patches.html`
* Patch 提交 checklist：`https://docs.kernel.org/process/submit-checklist.html`
* ftrace：`https://docs.kernel.org/trace/ftrace.html`
* Tracer debugging：`https://docs.kernel.org/trace/debugging.html`
* Tracepoints：`https://docs.kernel.org/trace/tracepoints.html`
* Event tracing：`https://docs.kernel.org/trace/events.html`
* kdump：`https://docs.kernel.org/admin-guide/kdump/kdump.html`
* blk-mq：`https://docs.kernel.org/block/blk-mq.html`
* Queue sysfs：`https://docs.kernel.org/block/queue-sysfs.html`
* GFP masks from FS/IO context：`https://docs.kernel.org/core-api/gfp_mask-from-fs-io.html`

## 1. 问题分类与分层定位

### 1.1 问题类型分类

| 类型 | 典型症状 | 严重程度 | 处理优先级 |
|---|---|---|---|
| **IO 性能问题** | 延迟抖动、吞吐下降、尾延迟高 | 中 | P2 |
| **IO 错误** | EIO、介质错误、超时、reset | 高 | P1 |
| **数据损坏** | 文件内容错误、metadata 损坏、fsck 失败 | 最高 | P0 |
| **崩溃类** | Oops、Panic、vmcore | 最高 | P0 |
| **挂起类** | Hang、死锁、lockup、D 状态堆积 | 高 | P1 |
| **内存问题** | 泄漏、UAF、越界访问、double free | 高 | P1 |

### 1.2 存储栈分层

```
用户态 syscall
    ↓
VFS (Virtual File System)
    ↓
文件系统 (ext4/xfs/btrfs/f2fs)
    ↓
Page Cache / Writeback
    ↓
Block Layer (blk-mq, IO Scheduler)
    ↓
Driver (NVMe/SCSI/DM)
    ↓
Hardware (Device/Firmware)
```

定位问题时必须明确问题发生在哪一层，不同层使用不同的观测手段。

## 2. IO Slow / Tail Latency

### 2.1 基线建立

在分析 IO 慢之前，必须建立稳定的基线：

```bash
# 确认裸盘能力，仅在专用测试设备上执行
fio --name=baseline --filename=/dev/nvme0n1 --ioengine=io_uring \
    --bs=4k --rw=randread --iodepth=32 --numjobs=1 --time_based --runtime=60

# 当前配置
uname -a
cat /sys/block/nvme0n1/queue/scheduler
cat /sys/block/nvme0n1/queue/nr_requests
cat /sys/block/nvme0n1/queue/read_ahead_kb
cat /sys/block/nvme0n1/queue/max_sectors_kb
cat /sys/block/nvme0n1/queue/rq_affinity
cat /sys/block/nvme0n1/queue/write_cache
lscpu
numactl -H
cat /proc/irq/*/smp_affinity
```

### 2.2 宏观瓶颈判断

```bash
# 系统层观测
iostat -xmt 1
vmstat 1
mpstat -P ALL 1
cat /proc/pressure/*

# CPU 瓶颈判断
perf top -g
perf record -g -a -- sleep 10

# 内存/回收判断
cat /proc/meminfo
cat /proc/vmstat | grep -E "pgscan|allocstall|compact"
```

### 2.3 IO 链路拆解

使用 blktrace 拆解 IO 延迟在各层的分布：

```bash
# 采集 blktrace
blktrace -d /dev/nvme0n1 -o - | blkparse -i - -o trace.txt

# 使用 btt 分析延迟分布
blkparse -i nvme0n1 -d nvme0n1.dat
btt -i nvme0n1.dat -l latency.log

# 延迟分量：
# Q2G: Queue -> Get request (分配 request 的时间)
# G2I: Get -> Issue (插入调度器到下发的时间)
# I2D: Issue -> Driver (驱动处理时间)
# D2C: Driver -> Complete (设备执行时间)
```

`blk-mq` 相关结论必须区分：

* bio 是否已经进入 block layer
* request 是否被 plug/merge/scheduler 延迟
* 是否缺 tag、卡在 `hctx->dispatch`、driver `queue_rq()` 返回 busy，或设备侧不完成
* completion 是否已经到达但上层 endio/workqueue 未及时运行

不要假设 request 按提交顺序完成；若上层依赖顺序，必须指出具体保证机制。

### 2.4 区分真 IO 慢和伪 IO 慢

**伪 IO 慢的常见原因：**

1. **CPU 调度延迟**
   ```bash
   # 检查 CPU 调度延迟
   perf sched record -- sleep 10
   perf sched latency
   
   # 检查 runqueue 延迟
   bpftrace -e 'tracepoint:sched:sched_wakeup { @wakeups[args->pid] = count(); }'
   ```

2. **Direct Reclaim / Compaction**
   ```bash
   # 检查内存回收
   cat /proc/vmstat | grep -E "pgscan|allocstall"
   bpftrace -e 'kprobe:try_to_free_pages { @reclaim[pid] = count(); }'
   ```

3. **锁竞争**
   ```bash
   # 检查锁等待
   perf lock record -- sleep 10
   perf lock report
   
   # 或使用 bpftrace
   bpftrace -e 'fexit:mutex_lock { if (retval != 0) @contended[func] = count(); }'
   ```

### 2.5 tracepoint 观测

```bash
# Block layer
trace-cmd record -e block:block_rq_issue -e block:block_rq_complete \
    -e block:block_rq_insert -p function_graph -a -- sleep 10

# Writeback
trace-cmd record -e writeback:writeback_start -e writeback:writeback_written \
    -e writeback:wbc_writepage -- sleep 10

# 文件系统 (ext4 示例)
trace-cmd record -e ext4:ext4_writepage -e ext4:ext4_writepages \
    -e ext4:ext4_sync_file -e ext4:ext4_da_write_begin -- sleep 10
```

## 3. IO Error / Data Corruption

### 3.1 现场保护原则

数据损坏场景必须先保护证据：

```bash
# 1. 只读挂载
mount -o remount,ro /mount/point

# 2. 记录 SMART/NVMe 日志
smartctl -a /dev/nvme0n1
nvme get-log /dev/nvme0n1 --log-id=0x02  # Smart log
nvme get-log /dev/nvme0n1 --log-id=0x03  # Error log

# 3. SCSI sense (针对 SCSI 设备)
sg_logs /dev/sda

# 4. 如果可能，镜像备份
dd if=/dev/nvme0n1 of=/backup/nvme0n1.img bs=1M status=progress
```

### 3.2 错误来源定位

| 错误来源 | 症状 | 定位方法 |
|---|---|---|
| **应用写入错误** | 特定文件损坏 | 检查应用逻辑、syscall 返回值 |
| **Page Cache/Invalidation** | 读到旧数据/丢失 | trace `invalidate_inode_pages2`、`filemap_write_and_wait` |
| **FS Journal/Log** | 崩溃恢复后损坏 | 分析 journal replay、dm-log-writes |
| **Block Remap** | LBA 映射错误 | 检查 discard/fallocate/remap 路径 |
| **Driver Timeout/Reset** | NVMe/SCSI reset 后 IO 失败 | 检查 dmesg 中的 timeout/reset 日志 |
| **硬件介质错误** | SMART 错误、UNC 错误 | smartctl、NVMe error log |

### 3.3 构造最小复现

```bash
# 使用 dm-flakey 模拟设备故障
dmsetup create flakey --table "0 $SIZE flakey 1 /dev/nvme0n1 0 5 1"

# 使用 dm-log-writes 记录写入
dmsetup create log-writes --table "0 $SIZE log-writes /dev/nvme0n1 /tmp/log.bin"

# fault injection
echo 1 > /sys/block/nvme0n1/make-it-fail
echo 100 > /sys/kernel/debug/fail_make_request/probability
```

### 3.4 一致性语义验证

必须说明以下机制在问题路径中的作用：

* **Buffered I/O vs Direct I/O**: 页缓存的影响
* **fsync/fdatasync**: 持久化语义
* **REQ_PREFLUSH/FUA**: 原子写入保证
* **Journal/Log**: 崩溃一致性

数据损坏结论必须区分"持久化语义未满足"、"读到旧数据"、"metadata 损坏"、"设备返回错误数据"和"测试程序未校验写入返回值"。缺少 fsync/flush 语义时，只能说明 workload 不保证崩溃后一致，不能直接归因为内核损坏。

## 4. Oops / Panic / Crash

### 4.1 vmcore 采集与分析

```bash
# 确认 kdump 配置
cat /proc/cmdline | grep crashkernel
systemctl status kdump

# 分析 vmcore
crash /usr/lib/debug/lib/modules/$(uname -r)/vmlinux /var/crash/vmcore

# 常用 crash 命令
crash> bt -a          # 所有 CPU 栈
crash> log            # dmesg 日志
crash> ps             # 进程列表
crash> files          # 打开的文件
crash> mount          # 挂载点
crash> struct task_struct <addr>  # 查看结构体
```

### 4.2 符号化 Call Trace

```bash
# 使用 decode_stacktrace.sh
dmesg | /usr/src/linux/scripts/decode_stacktrace.sh /usr/lib/debug/lib/modules/$(uname -r)/vmlinux

# 使用 faddr2line
/usr/src/linux/scripts/faddr2line vmlinux function+0x123/0x456
```

### 4.3 分析要点

1. **RIP 和寄存器**：确定崩溃指令和操作数
2. **Fault Address**：访问的非法地址
3. **当前进程上下文**：`current`、进程状态、锁状态
4. **中断上下文**：是否在中断、软中断、RCU read-side
5. **对象生命周期**：访问的对象是否已释放

### 4.4 常见崩溃原因

| 崩溃类型 | 典型原因 | 定位方法 |
|---|---|---|
| **NULL pointer dereference** | 未初始化指针、错误路径 | 检查 RIP 附近的指针来源 |
| **Invalid opcode** | 函数指针错误、栈损坏 | 检查函数指针设置点 |
| **General protection fault** | 非法内存访问、对齐错误 | 检查内存操作和对齐 |
| **Page fault** | 访问已释放内存、越界 | 结合 KASAN 定位 |
| **Divide error** | 除零 | 检查除数来源 |

## 5. Hang / Deadlock / Lockup

### 5.1 SysRq 信息采集

**第一时间采集，不要延迟：**

```bash
# SysRq 组合键 (物理机或带外控制台)
Alt + SysRq + t    # 所有任务栈
Alt + SysRq + w    # D 状态任务
Alt + SysRq + l    # 所有 CPU 栈

# 或通过 /proc 触发
echo t > /proc/sysrq-trigger
echo w > /proc/sysrq-trigger
echo l > /proc/sysrq-trigger
```

### 5.2 区分 Hang 类型

| 类型 | 症状 | 检查方法 |
|---|---|---|
| **Hung Task** | 进程 D 状态超时 | `cat /proc/*/stack`、hung task watchdog |
| **RCU Stall** | RCU grace period 不推进 | dmesg 中的 RCU stall 信息 |
| **Soft Lockup** | CPU 长时间不调度 | `cat /proc/softirqs`、watchdog |
| **Hard Lockup** | CPU 完全无响应 | NMI watchdog、带外心跳 |
| **IO Hang** | IO 长期不完成 | blk-mq inflight、tag 状态 |
| **锁死锁** | 多线程相互等待 | lockdep、持锁者栈 |

### 5.3 锁问题分析

```bash
# lockdep 信息
cat /proc/lockdep
cat /proc/lockdep_stats
dmesg | grep -i lockdep

# perf lock 分析
perf lock record -- sleep 10
perf lock report

# 查找持锁者
bpftrace -e '
kprobe:mutex_lock { @lock[arg0] = pid; }
kretprobe:mutex_lock /@lock[arg0]/ { printf("PID %d acquired lock %p\n", pid, arg0); delete(@lock[arg0]); }
'
```

### 5.4 blk-mq Hang 分析

```bash
# 检查 inflight IO
cat /sys/block/nvme0n1/inflight

# 检查 blk-mq tag 状态
ls /sys/kernel/debug/block/nvme0n1/hctx*/
cat /sys/kernel/debug/block/nvme0n1/hctx0/tags
cat /sys/kernel/debug/block/nvme0n1/hctx0/sched_tags
cat /sys/kernel/debug/block/nvme0n1/hctx0/busy
cat /sys/kernel/debug/block/nvme0n1/hctx0/dispatch

# 检查 NVMe queue
cat /sys/kernel/debug/nvme0n1/qid
```

## 6. Memory Bugs

### 6.1 工具选择

| 问题类型 | 工具 | Kconfig |
|---|---|---|
| **UAF/越界访问** | KASAN / KFENCE | `CONFIG_KASAN`, `CONFIG_KFENCE` |
| **未初始化读** | KMSAN | `CONFIG_KMSAN` |
| **数据竞争** | KCSAN | `CONFIG_KCSAN` |
| **内存泄漏** | kmemleak | `CONFIG_DEBUG_KMEMLEAK` |
| **锁验证** | lockdep | `CONFIG_PROVE_LOCKING` |

### 6.2 KASAN 使用

```bash
# 启用 KASAN (需要重新编译内核)
# Kernel hacking -> Memory debugging -> KASAN

# 查看 KASAN 报告
dmesg | grep -i kasan

# 典型 KASAN 输出分析：
# BUG: KASAN: use-after-free in function+0x123/0x456
# Read of size 4 at addr ffff88812345678 by task test/1234
# Freed by task test/1235:
#  [<ffffffff81234567>] kfree+0x67/0x180
#  [<ffffffffa0123456>] module_function+0x56/0x78
```

### 6.3 生命周期检查清单

* 对象分配点
* 对象发布点（对其他线程可见）
* 引用计数操作
* RCU grace period
* workqueue/timer 回调访问
* 错误路径释放

### 6.4 常见内存问题模式

| 模式 | 症状 | 根因 |
|---|---|---|
| **UAF** | 访问已释放内存 | 引用计数错误、RCU 误用 |
| **Double Free** | 重复释放 | 错误路径多次释放 |
| **越界访问** | 访问数组边界外 | 索引未检查、size 计算错误 |
| **未初始化读** | 使用随机值 | 栈变量未初始化 |
| **数据竞争** | 并发访问无同步 | 缺少锁或内存屏障 |

## 7. 常用观测工具速查

### 7.1 系统层

```bash
iostat -xmt 1          # IO 统计
vmstat 1               # 内存/进程统计
mpstat -P ALL 1        # 每 CPU 统计
cat /proc/pressure/*   # PSI 压力指标
```

### 7.2 Trace

```bash
trace-cmd record -e <event> -- sleep 10
perf record -e <event> -a -- sleep 10
bpftrace -e 'tracepoint:... { ... }'
```

### 7.3 Block Layer

```bash
blktrace -d /dev/nvme0n1 -o - | blkparse -i -
btt -i blktrace.dat
```

### 7.4 Crash 分析

```bash
crash vmlinux vmcore
drgn vmlinux vmcore
```

## 8. 输出要求

问题定位的输出必须包含：

1. **现象描述**：明确的问题定义和影响范围
2. **环境信息**：内核版本、config、硬件、workload
3. **假设列表**：2-4 个可证伪假设
4. **验证过程**：每个假设的观测动作和结果
5. **根因定位**：代码路径、数据结构、触发条件
6. **修复方案**：临时规避、根治 patch
7. **验证结果**：reproducer 不再触发、回归测试通过

所有结论必须有证据支撑，不能凭猜测。
