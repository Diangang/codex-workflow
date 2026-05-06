# Performance Optimization Reference

本文件为内核存储与 I/O 子系统的性能优化工作提供详细方法论。适用场景：IO 吞吐、p99/p99.9 尾延迟、锁竞争、CPU 开销、队列调度、内存回收干扰。

## Primary References

* ftrace：`https://docs.kernel.org/trace/ftrace.html`
* blk-mq：`https://docs.kernel.org/block/blk-mq.html`
* Queue sysfs：`https://docs.kernel.org/block/queue-sysfs.html`
* Memory allocation：`https://docs.kernel.org/core-api/memory-allocation.html`
* GFP masks from FS/IO context：`https://docs.kernel.org/core-api/gfp_mask-from-fs-io.html`
* fio：`https://fio.readthedocs.io/en/latest/fio_doc.html`

## 1. 性能优化方法论

### 1.1 核心流程

```
定义指标 → 建立基线 → 宏观定位 → 微观剖析 → 因果确认 → 设计变更 → A/B 验证
```

**每一步都不可跳过。** 没有基线不开始优化，没有因果确认不写 patch，没有 A/B 验证不提交。

### 1.2 指标定义

在开始任何优化前，必须明确定义要优化的指标：

| 指标类型 | 具体指标 | 采集方法 |
|---|---|---|
| **吞吐** | IOPS、BW (MB/s) | fio、iostat |
| **延迟** | p50/p90/p95/p99/p99.9/p99.99 | fio latency percentiles |
| **CPU** | CPU usage、上下文切换 | mpstat、perf stat |
| **调度** | runqueue latency、off-CPU time | perf sched、offcputime |
| **锁** | lock wait time、contention count | perf lock |
| **IO 链路** | Q2G/G2I/I2D/D2C | blktrace/btt |

### 1.3 性能优化红线

* **不用单轮测试或平均延迟证明性能收益**
* **不牺牲数据一致性、错误处理或 ABI 兼容性换性能**
* **硬件已达上限时，不把问题包装成内核优化**
* **不在没有基线的情况下开始优化**
* **不在无法解释因果关系时提交 patch**

## 2. 基线建立

### 2.1 环境固定

```bash
# CPU 频率固定
cpupower frequency-set -g performance
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# NUMA/CPU pinning
numactl -H
taskset -c 0-7 <workload>

# IRQ affinity
cat /proc/interrupts | grep nvme
cat /proc/irq/<irq>/smp_affinity_list

# IO Scheduler
cat /sys/block/nvme0n1/queue/scheduler
echo none > /sys/block/nvme0n1/queue/scheduler

# Queue depth
cat /sys/block/nvme0n1/queue/nr_requests
echo 128 > /sys/block/nvme0n1/queue/nr_requests

# 功耗策略
cat /sys/module/pcie_aspm/parameters/policy
cat /sys/devices/system/cpu/intel_pstate/no_turbo
```

修改 scheduler、`nr_requests`、IRQ affinity、CPU governor 等参数前，先记录原值和恢复命令；这些是实验控制变量，不应直接作为根因修复结论。

### 2.2 基线 workload

```bash
# 基线测试 - 必须多轮运行，仅在专用测试设备上执行
for i in $(seq 1 5); do
    fio --name=baseline-$i --filename=/dev/nvme0n1 --ioengine=io_uring \
        --bs=4k --rw=randread --iodepth=32 --numjobs=4 \
        --time_based --runtime=120 --ramp_time=30 \
        --lat_percentiles=1 --percentile_list=50:90:95:99:99.5:99.9:99.99 \
        --output-format=json+ --output=baseline-$i.json
done
```

### 2.3 基线验证

多轮运行结果波动应 < 5%。若波动过大：

1. 检查是否有后台干扰
2. 检查温度限流
3. 检查 GC/trim 干扰
4. 增加预热时间

## 3. 宏观瓶颈定位

### 3.1 瓶颈分类

| 瓶颈类型 | 典型症状 | 关键指标 |
|---|---|---|
| **CPU bound** | CPU 100%、workload scale 不上去 | perf top 热点集中 |
| **IO bound** | 设备利用率 100%、await 高 | iostat %util、D2C |
| **内存/回收** | allocstall、pgscan 高 | /proc/vmstat |
| **调度** | 大量上下文切换、runqueue 深 | perf sched latency |
| **锁竞争** | off-CPU 时间集中在锁 | perf lock contention |
| **设备/固件** | D2C 长、设备内部排队 | blktrace D2C、NVMe log |

### 3.2 宏观观测

```bash
# IO 层面
iostat -xmt 1 10
cat /proc/diskstats

# CPU 层面
mpstat -P ALL 1 10
perf stat -a -- sleep 10

# 内存/回收层面
vmstat 1 10
cat /proc/vmstat | grep -E "pgscan|pgsteal|allocstall|compact"
cat /proc/pressure/*

# 系统调度
perf sched record -- sleep 10
perf sched latency --sort max
```

## 4. 微观剖析

### 4.1 On-CPU 火焰图

```bash
# 采集
perf record -F 99 -g -a -- sleep 30

# 生成火焰图
perf script | stackcollapse-perf.pl | flamegraph.pl > on-cpu.svg

# 或使用 bpftrace
bpftrace -e 'profile:hz:99 { @[kstack] = count(); }' > /tmp/stacks.out
```

### 4.2 Off-CPU 火焰图

```bash
# 使用 bpftrace
bpftrace -e '
kprobe:finish_task_switch {
    @start[tid] = nsecs;
}
kretprobe:finish_task_switch /@start[tid]/ {
    @off_cpu[kstack, tid] = sum(nsecs - @start[tid]);
    delete(@start[tid]);
}
'

# 或使用 offcputime (BCC 工具)
offcputime -f 30 > /tmp/offcpu.stacks
flamegraph.pl --color=io < /tmp/offcpu.stacks > offcpu.svg
```

### 4.3 锁分析

```bash
# perf lock
perf lock record -- sleep 10
perf lock report --sort acquired,contended,wait_total

# lock contention (BCC)
lockcontention.py --duration 10

# bpftrace 定向追踪
bpftrace -e '
kprobe:mutex_lock { @start[tid] = nsecs; }
kretprobe:mutex_lock /@start[tid]/ {
    $lat = nsecs - @start[tid];
    if ($lat > 1000000) {  // > 1ms
        printf("mutex_lock took %d us at %s\n", $lat/1000, kstack);
    }
    @latency = hist($lat);
    delete(@start[tid]);
}
'
```

### 4.4 Block Layer 剖析

```bash
# blktrace 完整采集
blktrace -d /dev/nvme0n1 -w 30
blkparse -i nvme0n1 -d nvme0n1.dat
btt -i nvme0n1.dat

# Q2G: 分配 request 延迟 (可能受锁影响)
# G2I: 插入调度器延迟
# I2D: 调度器到下发延迟
# D2C: 设备执行延迟

# blk-mq 状态
cat /sys/kernel/debug/block/nvme0n1/state
cat /sys/kernel/debug/block/nvme0n1/hctx*/type
cat /sys/kernel/debug/block/nvme0n1/hctx*/cpu_list

# IO 延迟分布
bpftrace -e '
tracepoint:block:block_rq_issue { @start[args->dev, args->sector] = nsecs; }
tracepoint:block:block_rq_complete /@start[args->dev, args->sector]/ {
    @latency = hist(nsecs - @start[args->dev, args->sector]);
    delete(@start[args->dev, args->sector]);
}
'
```

### 4.5 Function Graph Trace

```bash
# 追踪特定函数的执行时间
echo ext4_writepages > /sys/kernel/debug/tracing/set_graph_function
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
sleep 5
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace

# 或使用 trace-cmd
trace-cmd record -p function_graph -g ext4_writepages -- sleep 5
trace-cmd report
```

## 5. 常见优化方向

### 5.1 减少锁竞争

| 优化手段 | 适用场景 | 注意事项 |
|---|---|---|
| **锁粒度拆分** | 单一大锁瓶颈 | 注意锁序和死锁 |
| **per-cpu 化** | 全局计数器、统计 | 读取需要聚合 |
| **RCU 替换** | 读多写少 | RCU grace period 延迟 |
| **lock-free** | 简单数据结构 | 内存屏障正确性 |
| **trylock + 批量** | 高竞争热路径 | 退避策略 |

### 5.2 减少 IO 路径开销

| 优化手段 | 适用场景 | 注意事项 |
|---|---|---|
| **batch submission** | 大量小 IO | 延迟 vs 吞吐权衡 |
| **减少 bio split** | 大 IO 分段多 | 硬件限制 |
| **inline crypto** | 加密 IO | 硬件支持 |
| **避免不必要的 flush/FUA** | 非必要持久化 | 数据一致性 |
| **IO 合并优化** | 相邻小 IO | scheduler 选择 |

### 5.3 减少内存/回收干扰

| 优化手段 | 适用场景 | 注意事项 |
|---|---|---|
| **预分配** | 热路径分配 | 内存占用 |
| **mempool** | 关键路径保证分配 | 池大小 |
| **控制热路径分配** | direct reclaim 或分配失败放大尾延迟 | 预分配、mempool、合适的 GFP scope；atomic 上下文不能睡眠 |
| **writeback 调优** | 脏页回写干扰 | dirty_ratio/dirty_expire |
| **cgroup memory 隔离** | 容器场景 | 配额和回收策略 |

### 5.4 减少 cacheline bounce

| 优化手段 | 适用场景 | 注意事项 |
|---|---|---|
| **____cacheline_aligned** | 热数据结构 | 内存浪费 |
| **per-cpu 变量** | 频繁更新的计数器 | 聚合开销 |
| **读写分离** | 混合读写字段 | 结构体重新布局 |
| **NUMA 亲和** | 跨 NUMA 访问 | 数据分布 |

## 6. A/B 验证规范

### 6.1 验证要求

```bash
# A 组 (baseline)
git checkout baseline-branch
make -j$(nproc)
# 运行 workload 5 轮

# B 组 (optimized)
git checkout optimized-branch
make -j$(nproc)
# 运行 workload 5 轮

# 对比
# 相同物理机、相同设备、相同环境
# 排除第一轮预热数据
# 使用中位数对比
```

### 6.2 验证矩阵

| 验证维度 | 内容 |
|---|---|
| **目标 workload** | 优化目标场景的性能提升 |
| **其他 workload** | 确认无回归 |
| **功能回归** | fstests/blktests 无新增失败 |
| **稳定性** | 长稳运行无异常 |
| **Debug config** | KASAN/KCSAN/lockdep 无告警 |
| **dmesg clean** | 无新增 WARN/BUG |

### 6.3 数据呈现

性能报告必须包含：

1. **完整百分位分布**：p50/p90/p95/p99/p99.5/p99.9/p99.99
2. **多轮结果**：每轮结果 + 中位数/标准差
3. **前后对比**：百分比提升/下降
4. **火焰图对比**：on-CPU/off-CPU before/after
5. **关键 trace 证据**：热路径变化

## 7. 尾延迟专项

### 7.1 尾延迟常见根因

| 根因 | 典型影响 | 排查方法 |
|---|---|---|
| **Direct Reclaim** | p99 抖动 | `allocstall`、`pgscan_direct` |
| **Journal Commit** | fsync p99 | ext4/jbd2 tracepoint |
| **IO Scheduler** | 请求等待 | blktrace I2D |
| **设备 GC/Trim** | 随机延迟脉冲 | D2C 分布、NVMe log |
| **NUMA 跨节点** | CPU cache miss | perf mem、c2c |
| **Interrupt Coalescing** | 批量完成延迟 | D2C 分布模式 |
| **锁竞争** | 个别 IO 被阻塞 | perf lock、off-CPU |
| **Cgroup throttle** | 限速触发 | blk-throttle tracepoint |

### 7.2 尾延迟观测

```bash
# 延迟分布 (bpftrace)
bpftrace -e '
tracepoint:block:block_rq_issue { @start[args->dev, args->sector] = nsecs; }
tracepoint:block:block_rq_complete /@start[args->dev, args->sector]/ {
    $lat = nsecs - @start[args->dev, args->sector];
    @lat_hist = hist($lat);
    if ($lat > 1000000) {  // > 1ms
        @slow_io = count();
        printf("Slow IO: %d us, %s\n", $lat/1000, kstack);
    }
    delete(@start[args->dev, args->sector]);
}
'

# 回收对延迟的影响
bpftrace -e '
kprobe:try_to_free_pages { @reclaim_start[tid] = nsecs; }
kretprobe:try_to_free_pages /@reclaim_start[tid]/ {
    @reclaim_lat = hist(nsecs - @reclaim_start[tid]);
    delete(@reclaim_start[tid]);
}
'
```

## 8. 输出要求

性能优化的输出必须包含：

1. **指标定义**：优化的具体目标
2. **基线数据**：多轮运行的完整百分位
3. **瓶颈分析**：宏观/微观定位过程和证据
4. **因果关系**：代码路径到性能热点的映射
5. **Patch/变更**：最小侵入的优化方案
6. **A/B 结果**：before/after 完整对比
7. **副作用评估**：回归测试、其他 workload 影响
8. **适用范围**：在什么条件下有效/无效
