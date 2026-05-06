# Documentation Templates

本文件保存正式文档模板和物理机操作约束。仅在用户要求生成 Design Doc、RCA、Debug Report、Test Plan、Test Report、OnCall 摘要，或需要登录真实物理机执行内核调试/测试时读取。

## 1. Common Documentation Rules

所有内核文档必须可追溯、可复现、可审查。

### Metadata

| 字段 | 要求 |
|---|---|
| 标题 | `[子系统] 简短主题` |
| 作者 / 日期 | 姓名、起止日期，多人协作列贡献者 |
| 内核版本 | `uname -a`、upstream base、内部/发行版分支 |
| 硬件 / 平台 | CPU、NUMA、盘型号、固件、HBA/RAID/NVMe/SCSI |
| 关键 config | `CONFIG_PREEMPT*`、`CONFIG_BLK*`、`CONFIG_*_FS`、debug config |
| 关联工单 / Patch | 工单、commit、邮件线程、测试报告 |
| 状态 | Draft / Under Review / Landed / Archived |

### Evidence Rules

* 源码引用使用 `path/to/file.c:function()` 或 `path/to/file.c:Lx`，注明 commit。
* 命令、脚本、配置使用 fenced code block，且可直接执行。
* 日志/trace 保留时间戳、采集命令、采集环境和关键上下文。
* 明确区分 `[Fact]`、`[Inference]`、`[Hypothesis]`、`[Conclusion]`。
* 不能用"疑似"作为最终结论；缺数据写 `Pending: 需要采集 ...`。

## 2. Design Doc Template

适用：新特性设计、子系统重构、接口变更、RFC。

```text
1. Background & Motivation
   - 当前机制和数据路径
   - 问题或新增诉求
   - Non-goals

2. Proposed Design
   - 总体思路
   - 数据路径变更前后对比
   - 关键数据结构和生命周期
   - 锁、上下文、内存分配语义
   - 与 VFS/MM/block/cgroup/PSI 等交互

3. Alternatives Considered
   - 替代方案
   - 舍弃原因和 trade-off

4. Compatibility & Risk
   - ABI/API/on-disk format/sysfs/ioctl 影响
   - 回滚方案
   - 性能和稳定性风险

5. Test Plan
   - KUnit/kselftest/fstests/blktests/fio
   - fault injection
   - KASAN/KCSAN/lockdep/syzkaller

6. Patch Series Outline
   - 每个 patch 的职责与依赖

7. Open Questions
```

质量红线：

* 复杂链路必须有数据路径图或调用链
* 锁序变更必须列出旧锁序和新锁序，并说明 lockdep 验证
* 新增 `GFP_*`、睡眠点、中断上下文假设必须显式声明
* 性能声明必须有基准数据

## 3. Debug Report / RCA Template

适用：线上故障、Oops/Panic/Hang、数据损坏、性能抖动、回归问题。

```text
1. Incident Summary
   - 现象、影响范围、根因、修复动作、是否止损

2. Environment
   - 集群/机型/盘型号/固件/内核版本/config
   - 发生时间窗口和影响实例数

3. Symptoms & Evidence
   - 监控曲线
   - dmesg/Oops/Call Trace
   - perf/ftrace/eBPF/blktrace
   - 每条证据的采集命令和文件路径

4. Timeline
   - 告警、介入、现场留存、止损、定位、修复

5. Root Cause Analysis
   - 假设列表与证伪方法
   - 验证过程
   - 因果链：触发条件 -> 代码路径 -> 数据结构异常 -> 外部现象
   - 为什么此前未被发现

6. Fix / Workaround
   - 临时规避
   - 根治 patch
   - 回归验证

7. Follow-ups
   - 监控、测试、上游回合、影响面排查
```

特殊要求：

* Crash/Panic：包含符号化 Call Trace、`bt -a`、`log`、`ps`、关键结构体输出
* Hang/Lockup：包含 SysRq `t/w/l`、D 状态任务、持锁者 PID 和栈、blk-mq inflight/tag 快照
* 性能抖动：包含基线、异常、恢复后三组数据，以及 on-CPU/off-CPU、PSI、blktrace/btt 对比

## 4. Test Plan Template

```text
1. Test Objective
   - 正确性 / 性能 / 稳定性 / 故障恢复 / 兼容性
   - Non-goals

2. Scope & Matrix
   - 内核版本 x FS x 挂载参数 x 调度器 x 盘型号

3. Workload Model
   - fio job / reproducer / syzkaller config
   - 并发度、队列深度、块大小、读写比、持续时间

4. Fault Injection Plan
   - 注入点、概率、时机、预期路径

5. Success Criteria
   - 定量指标
   - 无 WARN/BUG/KASAN/lockdep
   - 数据一致性和 fsck 结果

6. Monitoring & Data Collection
   - dmesg、perf、trace、blktrace、sysfs/debugfs

7. Risk & Rollback
```

## 5. Test Report Template

```text
1. Executive Summary
   - Pass / Fail / Blocked

2. Environment Snapshot
   - uname、config diff、硬件、固件、调度器、mount、lsblk

3. Test Matrix & Results
   - 用例、结果、指标、日志路径

4. Performance Results
   - IOPS、BW、p50/p95/p99/p99.9
   - baseline 对比

5. Stability Results
   - 运行时长
   - WARN/BUG/KASAN/lockdep/hung task

6. Fault Injection Results
   - 每个注入点的预期路径和实际结果

7. Known Issues & Exemptions

8. Attachments
```

质量红线：

* 性能数据必须有完整百分位分布
* Pass 结论必须能从附件中的命令完整重放
* KASAN/KCSAN/lockdep 告警即 Fail，必须关联 Debug Report
* 故障注入不是"不崩即 pass"，必须验证错误路径清理

## 6. Physical Machine Operation SOP

适用：登录真实物理机进行内核开发、调试、测试、压测、故障注入。

### Pre-login

* 确认机器归属：测试机、公共开发机、生产机
* 确认可破坏范围：是否允许重启、改内核、写裸盘、跑压力
* 确认带外管理：BMC/IPMI/iLO/SOL 可用
* 若是问题现场，先保存当前 dmesg、监控、sysfs/debugfs 快照

### Day-1 Health Check

```bash
dmesg -T | grep -iE "error|warn|fail|mce|nvme|scsi"
uptime
top -bn1 | head -20
iostat -xmt 1 3
systemctl status kdump
df -h / /boot /var/crash
```

### Dangerous Operations

* 编译内核时按机器用途控制 `make -j`，共享机使用 `nice`/`ionice`
* 加载测试模块前确认 init/exit、错误路径、引用计数和日志开关
* `mkfs`、裸盘 `fio`、`dd`、dm fault injection 前反复确认目标盘：`lsblk`、`blkid`、挂载点
* 压测期间持续监控 dmesg、温度、SMART/NVMe error log、inflight

### Teardown

* 停止 `fio`、`stress-ng`、自研压测进程
* 卸载临时文件系统，移除测试模块
* 还原 sysctl、sysfs queue 参数、CPU governor、IRQ affinity
* 新内核重启后通过带外控制台确认启动和根文件系统挂载；失败时保留 Oops/vmcore
