# Testing Reference

本文件为内核存储与 I/O 子系统的测试工作提供详细方法论。涵盖内核自测、存储回归测试、压力测试、性能测试、故障注入。

## Primary References

* kselftest：`https://docs.kernel.org/dev-tools/kselftest.html`
* KUnit：`https://docs.kernel.org/dev-tools/kunit/index.html`
* Fault injection：`https://docs.kernel.org/fault-injection/fault-injection.html`
* fstests：`git://git.kernel.org/pub/scm/fs/xfs/xfstests-dev.git`
* blktests：`https://github.com/linux-blktests/blktests`
* fio：`https://fio.readthedocs.io/en/latest/fio_doc.html`

## 1. 测试分类与目标

### 1.1 测试类型矩阵

| 类型 | 目标 | 工具 | 场景 |
|---|---|---|---|
| **内核自测** | 功能正确性 | KUnit、kselftest、LTP | patch 验证、CI |
| **存储回归测试** | 子系统回归 | fstests、blktests | 新内核验证、backport 后验证 |
| **压力测试** | 稳定性、并发竞态 | fsstress、stress-ng、syzkaller | 长稳、竞态放大 |
| **性能测试** | 性能基线和回归 | fio、filebench | 性能优化验证、版本对比 |
| **故障注入** | 错误路径正确性 | fail_make_request、dm-flakey | 容错能力、崩溃恢复 |

### 1.2 测试意图明确化

开始任何测试前必须回答：

1. **测试什么**：具体的功能点、代码路径、场景
2. **为什么测**：验证修复、防止回归、压力覆盖
3. **怎么判断**：明确的成功/失败标准
4. **测多久**：时间约束、覆盖要求

### 1.3 安全前置

* `fstests` 的 `SCRATCH_DEV`、`blktests` 的 `TEST_DEVS`、裸盘 `fio`、`mkfs`、`dmsetup`、fault injection 都可能破坏数据；执行前必须用 `lsblk -f`、`blkid`、`findmnt` 确认目标盘不是生产盘或含数据盘。
* 测试报告中记录目标盘确认命令和输出摘要；不能只写 `/dev/nvme0n1`。
* 在共享机器上运行长稳、fio 或 syzkaller 前，确认 CPU/IO/内存配额和停止方式。
* 对故障注入，先写明预期错误路径和清理标准，再执行注入。

## 2. 内核自测

### 2.1 KUnit

适用于函数级/模块级白盒测试。KUnit 是 white-box 测试，适合覆盖内部函数和错误路径；它不能替代真实设备或完整 FS/block 集成测试。

```bash
# 运行所有 KUnit 测试
./tools/testing/kunit/kunit.py run

# 运行特定子系统测试
./tools/testing/kunit/kunit.py run --kunitconfig=fs/ext4/.kunitconfig

# 运行特定测试套件
./tools/testing/kunit/kunit.py run <suite_name>

# 带 UML (User Mode Linux) 运行
./tools/testing/kunit/kunit.py run --arch=um
```

### 2.2 kselftest

适用于用户态可触发的内核接口回归测试。kselftest 通常在构建、安装并启动目标内核后运行，适合验证 syscall、uapi 和用户可见行为。

```bash
# 编译 kselftest
make -C tools/testing/selftests TARGETS=<subsystem>

# 运行特定子系统
make -C tools/testing/selftests TARGETS=filesystems run_tests
make -C tools/testing/selftests TARGETS=io_uring run_tests

# 安装到目标系统
make -C tools/testing/selftests TARGETS=<subsystem> INSTALL_PATH=<path> install
```

### 2.3 LTP (Linux Test Project)

适用于系统调用和 POSIX 合规性测试。

```bash
# 编译安装
./configure && make && make install

# 运行 IO 相关测试
cd /opt/ltp
./runltp -f io -l result.log

# 运行文件系统相关测试
./runltp -f fs -l result.log
```

## 3. 存储回归测试

### 3.1 fstests (xfstests)

文件系统正确性、fsync、崩溃恢复、压力回归测试。

```bash
# 配置 local.config
cat > local.config <<EOF
export TEST_DEV=/dev/nvme0n1p1
export TEST_DIR=/mnt/test
export SCRATCH_DEV=/dev/nvme0n1p2
export SCRATCH_MNT=/mnt/scratch
export FSTYP=ext4
export MKFS_OPTIONS="-O 64bit"
export MOUNT_OPTIONS="-o discard"
EOF

# 运行所有测试
./check

# 运行特定测试组
./check -g quick
./check -g auto

# 运行特定测试
./check generic/001 ext4/001

# 排除已知失败
./check -E exclude_list

# 多文件系统覆盖
for fs in ext4 xfs btrfs; do
    export FSTYP=$fs
    ./check -g auto 2>&1 | tee results-$fs.log
done
```

**关键测试组：**

| 测试组 | 覆盖 |
|---|---|
| `quick` | 快速回归，基本功能 |
| `auto` | 完整自动化测试 |
| `dangerous` | 可能损坏设备或数据 |
| `shutdown` | 崩溃恢复相关 |
| `log` | 日志/journal 相关 |

### 3.2 blktests

块层、NVMe、SCSI、loop、DM、ublk 回归测试。

```bash
# 配置
cat > config <<EOF
TEST_DEVS=(/dev/nvme0n1)
EOF

# 再次确认 TEST_DEVS 中没有保留数据
lsblk -f /dev/nvme0n1
findmnt -S /dev/nvme0n1

# 运行所有测试
./check

# 运行特定子系统
./check block
./check nvme
./check scsi
./check dm

# 运行特定测试
./check block/001 nvme/001
```

### 3.3 回归测试策略

针对不同变更类型选择测试：

| 变更类型 | 必须测试 |
|---|---|
| **ext4 bugfix** | fstests generic/* + ext4/* |
| **xfs bugfix** | fstests generic/* + xfs/* |
| **block layer** | blktests block/* |
| **NVMe driver** | blktests nvme/* |
| **DM/多路径** | blktests dm/* |
| **VFS/通用路径** | fstests generic/* + 多 FS |

## 4. 压力测试

### 4.1 fsstress

文件系统压力和竞态放大。

```bash
# 基本压力测试
fsstress -d /mnt/test -n 100000 -p 20 -l 0

# 特定操作
fsstress -d /mnt/test -p 16 -n 50000 \
    -f write=10 -f read=10 -f creat=5 -f unlink=5 \
    -f fsync=3 -f fallocate=2 -f truncate=2

# 配合故障注入
fsstress -d /mnt/test -p 16 -n 0 &
# 同时执行 freeze/thaw 或 device fault
```

### 4.2 stress-ng

系统级压力测试。

```bash
# IO 压力
stress-ng --io 16 --hdd 8 --timeout 3600

# 内存压力
stress-ng --vm 4 --vm-bytes 80% --mmap 4 --timeout 3600

# 混合压力
stress-ng --io 8 --hdd 4 --vm 2 --cpu 4 --timeout 3600
```

### 4.3 syzkaller

内核 syscall fuzz 测试。

```bash
# 配置 syzkaller (syz-manager.cfg)
{
    "target": "linux/amd64",
    "http": "0.0.0.0:56741",
    "workdir": "/tmp/syzkaller/workdir",
    "kernel_obj": "/path/to/kernel/build",
    "image": "/path/to/image",
    "syzkaller": "/path/to/syzkaller",
    "procs": 8,
    "enable_syscalls": [
        "open", "read", "write", "close",
        "ioctl$BLKRRPART", "mount", "umount",
        "fallocate", "fsync", "fdatasync"
    ]
}
```

### 4.4 长稳测试要求

| 指标 | 标准 |
|---|---|
| 运行时长 | >= 24 小时 (关键路径 >= 72 小时) |
| 并发度 | >= 实际生产并发 |
| dmesg | 无 WARN/BUG/异常 |
| 内存 | 无持续增长 (kmemleak clean) |
| 性能 | 无持续退化趋势 |

## 5. 性能测试

### 5.1 fio 基准测试

```bash
# 顺序读
fio --name=seq-read --filename=/dev/nvme0n1 --ioengine=io_uring \
    --bs=128k --rw=read --iodepth=32 --numjobs=4 \
    --time_based --runtime=120 --group_reporting

# 随机读写
fio --name=rand-rw --filename=/dev/nvme0n1 --ioengine=io_uring \
    --bs=4k --rw=randrw --rwmixread=70 --iodepth=64 --numjobs=4 \
    --time_based --runtime=120 --group_reporting

# 延迟敏感测试
fio --name=latency --filename=/dev/nvme0n1 --ioengine=io_uring \
    --bs=4k --rw=randread --iodepth=1 --numjobs=1 \
    --time_based --runtime=120 --lat_percentiles=1 \
    --percentile_list=50:90:95:99:99.5:99.9:99.99

# 文件系统测试
fio --name=fs-rw --directory=/mnt/test --ioengine=io_uring \
    --bs=4k --rw=randrw --rwmixread=70 --iodepth=32 --numjobs=8 \
    --size=1G --time_based --runtime=120 --group_reporting
```

### 5.2 性能测试规范

* **预热**：至少运行 30 秒预热再开始采集数据
* **多轮运行**：至少 3 轮，排除离群值
* **完整百分位**：必须包含 p50/p90/p95/p99/p99.9/p99.99
* **环境固定**：CPU pinning、NUMA、scheduler、电源策略
* **JSON 输出**：`--output-format=json+` 保存完整数据

### 5.3 性能对比规范

A/B 对比必须：

1. 同一台物理机
2. 相同的设备和固件
3. 相同的 config（除目标变更外）
4. 相同的 workload 和参数
5. 相同的预热时间和运行时长
6. 多轮运行取中位数
7. 报告完整百分位分布

## 6. 故障注入测试

### 6.1 Block Layer Fault Injection

```bash
# 使 IO 随机失败
echo 1 > /sys/block/nvme0n1/make-it-fail
echo 100 > /sys/kernel/debug/fail_make_request/probability
echo 5 > /sys/kernel/debug/fail_make_request/times
echo -1 > /sys/kernel/debug/fail_make_request/interval
echo 2 > /sys/kernel/debug/fail_make_request/verbose

# 清除
echo 0 > /sys/block/nvme0n1/make-it-fail
echo 0 > /sys/kernel/debug/fail_make_request/probability
```

`probability=100` 是极高错误率，适合配合 `times` 或 `interval` 做定点验证；不要直接用于长时间共享环境压测。

### 6.2 Memory Fault Injection

```bash
# Slab 分配失败
echo 1 > /sys/kernel/debug/failslab/task-filter
echo 10 > /sys/kernel/debug/failslab/probability
echo 5 > /sys/kernel/debug/failslab/times

# Page 分配失败
echo 10 > /sys/kernel/debug/fail_page_alloc/probability
echo 5 > /sys/kernel/debug/fail_page_alloc/times
```

### 6.3 Device Mapper Fault Injection

```bash
# dm-flakey: 模拟设备间歇性故障
dmsetup create flakey --table \
    "0 $SECTORS flakey /dev/nvme0n1 0 5 1 1 drop_writes"
# 5 秒正常，1 秒故障

# dm-delay: 模拟设备延迟
dmsetup create delayed --table \
    "0 $SECTORS delay /dev/nvme0n1 0 100"
# 所有 IO 增加 100ms 延迟

# dm-log-writes: 记录写入用于崩溃恢复测试
dmsetup create log-writes --table \
    "0 $SECTORS log-writes /dev/nvme0n1 /dev/sdb"
```

### 6.4 NVMe Fault Injection

```bash
# NVMe timeout injection (需要 CONFIG_FAIL_IO_TIMEOUT)
echo 1 > /sys/kernel/debug/nvme0/fault_inject/status
echo 100 > /sys/kernel/debug/nvme0/fault_inject/probability
```

### 6.5 故障注入验证原则

故障注入测试不是"不崩即 pass"，必须验证：

1. **错误码传播正确**：上层收到合理的 errno
2. **资源释放完整**：无内存泄漏、无引用计数泄漏
3. **状态恢复正确**：错误后系统可继续正常工作
4. **日志清晰**：故障被正确记录
5. **数据完整性**：已确认的数据不丢失

## 7. 测试执行规范

### 7.1 环境采集

运行前必须保存：

```bash
uname -a
cat /proc/cmdline
diff .config baseline.config
lsblk -o NAME,SIZE,ROTA,SCHED,MODEL,SERIAL
mount | grep -E "ext4|xfs|btrfs"
lscpu
numactl -H
cat /sys/block/*/queue/scheduler
cat /sys/block/*/queue/nr_requests
dmesg -c > /dev/null  # 清空 dmesg baseline
```

### 7.2 运行中监控

```bash
# 持续监控 dmesg
dmesg -w > /tmp/test_dmesg.log &

# IO 统计
iostat -xmt 1 > /tmp/test_iostat.log &

# 内存监控
vmstat 1 > /tmp/test_vmstat.log &
```

### 7.3 运行后检查

```bash
# 检查 dmesg
dmesg | grep -iE "warn|bug|kasan|kcsan|lockdep|rcu|hung"

# 检查文件系统
fsck -n /dev/nvme0n1p1

# 检查设备状态
smartctl -a /dev/nvme0n1
nvme get-log /dev/nvme0n1 --log-id=0x03

# 检查内存泄漏
echo scan > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak
```

### 7.4 失败处理

* 失败用例必须关联 Debug/RCA，不用"偶发"豁免
* 能复现时保留最小 reproducer
* KASAN/KCSAN/lockdep 告警即 Fail
* 记录失败环境、日志、栈信息

## 8. 输出要求

### Test Plan 输出

1. 测试目标和 Non-goals
2. 测试矩阵（版本 × FS × 设备 × workload）
3. 工具选择和 job file/配置
4. 成功标准
5. 风险和回滚

### Test Report 输出

1. Executive Summary（Pass/Fail/Blocked）
2. 环境快照
3. 测试结果矩阵
4. 性能数据（完整百分位）
5. 稳定性结果
6. 故障注入结果
7. 已知问题和豁免
8. 原始日志路径
