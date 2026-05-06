---
name: kernel-storage-workflow
description: Linux 内核存储与 I/O 子系统日常工作流，涵盖开发实现、问题定位、Backport、测试验证、Patch Review、性能优化六大类。当用户涉及 kernel patch、IO 慢/异常、Oops/Panic/Hang、死锁、数据损坏、UAF/越界/泄漏、社区 bugfix/特性回合、fstests/blktests/fio、ftrace/perf/eBPF/crash/drgn、VFS/ext4/xfs/btrfs/Page Cache/Writeback/blk-mq/NVMe/SCSI/io_uring 等关键词时触发。
---

# Kernel Storage Workflow

该 skill 面向 Linux 内核存储与 I/O 子系统的日常工程工作。核心要求是：**基于数据路径、源码和运行证据做判断，先定位层级和因果链，再给修复、测试或 review 结论**。

## 工作分类与参考文件

按任务类型优先读取对应 reference，避免一次加载无关内容：

| 分类 | 触发场景 | 参考文件 |
|---|---|---|
| **开发实现** | 编写或修改 kernel patch、设计数据路径、改锁/生命周期/错误路径/API | [references/development.md](references/development.md) |
| **问题定位** | IO 慢/异常、Oops/Panic/Hang、死锁、数据损坏、内存泄漏、UAF、越界访问 | [references/problem-debugging.md](references/problem-debugging.md) |
| **Backport** | 社区 bugfix、stable patch、安全修复、特性 patch 回合 | [references/backport.md](references/backport.md) |
| **测试验证** | 内核自测、存储回归、压力测试、性能测试、故障注入 | [references/testing.md](references/testing.md) |
| **Patch Review** | 内部 patch、社区 patch、backport patch、性能优化 patch、bugfix patch | [references/patch-review.md](references/patch-review.md) |
| **性能优化** | IO 吞吐、尾延迟、锁竞争、CPU 开销、队列调度 | [references/performance-optimization.md](references/performance-optimization.md) |
| **文档模板** | Design Doc、RCA、Debug Report、Test Plan/Report、物理机操作 | [references/document-templates.md](references/document-templates.md) |

若任务只需要简短答复，先用本文件的分类规则回答；若需要执行、排查、写文档或审查 patch，必须读取对应 reference。

## 可选外部 Skill 协作

如果当前环境中安装了以下外部 skill，可按场景协作；若未安装，继续使用本 skill 的 reference，不要阻塞工作。

| 场景 | 外部 Skill | 说明 |
|---|---|---|
| crash/vmcore 分析 | `linux-kernel-crash-debug` | 使用 crash 工具调试内核崩溃，提供 crash 命令、vmcore 分析、Agent 安全执行模式 |
| 代码 Review | `sashiko` | Linux 内核多阶段 agentic code review 系统，9 阶段审查流程覆盖正确性/并发/安全/资源管理 |

## 全局原则

所有工作类别都应遵循以下原则：

* **分层定位：** 优先明确问题所在层级：用户态 syscall / VFS / 文件系统 / Page Cache / Writeback / Block Layer / IO Scheduler / NVMe/SCSI / 硬件。
* **源码先行：** 先在目标 kernel tree 中确认实际函数、结构体、config 和 commit；外部文档只作规则和背景，不替代当前源码。
* **版本敏感：** 先确认 `uname -a`、upstream base、`.config`、发行版或内部补丁分支。内核行为与版本、config、厂商分支强相关。
* **上下文安全：** 对每个新增调用点确认 process/atomic/IRQ/softirq/RCU 上下文、可睡眠性、锁序、GFP/reclaim 语义和对象生命周期。
* **现场优先：** Oops/Panic/Hang/数据损坏先保存 dmesg、vmcore、trace、sysfs/debugfs 快照；不要先重启或运行破坏性修复。
* **最小侵入：** 生产环境优先使用 eBPF、tracepoint、`/proc`、`/sys`、debugfs、perf 等只读观测手段。
* **证据闭环：** 每个结论必须说明代码路径、关键数据结构/状态、观测证据、验证方式。
* **社区先验：** 深入修复前查 upstream git log、stable tree、LKML/子系统邮件线程，确认是否已有修复或已知回归。
* **不泛化猜测：** 只列 2-4 个可证伪假设，每个假设给出下一步观测动作和预期结果。

## Universal Entry Checklist

任何任务开始前先收集这些元信息。缺失项不要用猜测补齐；要么向用户索取，要么给出采集命令。

* 内核版本：`uname -a`、upstream base、内部分支名、相关 commit。
* 关键 config：`.config` 中 `CONFIG_PREEMPT*`、`CONFIG_BLK*`、`CONFIG_*_FS`、`CONFIG_KASAN`、`CONFIG_KCSAN`、`CONFIG_PROVE_LOCKING`。
* 平台与设备：CPU/NUMA、内存、盘型号、固件、HBA/RAID/NVMe/SCSI 拓扑、queue depth、IO scheduler。
* 负载与复现：业务 workload、syscall 或 fio/fsstress reproducer、复现概率、触发时长。
* 现场证据：`dmesg`、`journalctl -k`、vmcore、trace、perf、blktrace、监控曲线、sysfs/debugfs 快照。
* 影响范围：机器数、业务影响、是否仍在发生、是否可保留现场。

所有结论必须能回答四个问题：**哪个代码路径、哪个数据结构或状态、为什么产生该外部现象、用什么证据确认**。

## 1. 开发实现

**触发场景：** 编写或修改 kernel patch、设计存储数据路径、改锁/引用计数/错误路径、添加 tracepoint/sysfs/ioctl、实现 bugfix 或性能优化。

**参考文件：** [references/development.md](references/development.md)

**核心要点：**

* 先画调用链和对象生命周期，再改代码
* 每个新增睡眠点、分配点、锁、引用计数、work/timer 回调都要说明上下文
* FS/IO 路径避免随手使用 `GFP_NOFS`/`GFP_NOIO`；优先判断是否需要 `memalloc_nofs_save()` 或 `memalloc_noio_save()` scope
* blk-mq 修改必须理解 `bio`、`request`、`blk_mq_ctx`、`blk_mq_hw_ctx`、tag、queue freeze/quiesce 语义
* 对用户可见接口、tracepoint、on-disk format、sysfs/procfs 变更必须先评估兼容性
* patch 交付时带最小测试和错误路径验证，不用"能编译"替代行为验证

## 2. 问题定位

**触发场景：** IO 慢、IO error、Oops、Panic、Hang、死锁、数据损坏、内存泄漏、UAF、越界访问。

**参考文件：** [references/problem-debugging.md](references/problem-debugging.md)

**可选外部 Skill 协作：** 当前环境若安装了 `linux-kernel-crash-debug`，crash/vmcore 分析可调用该 skill；未安装时直接使用本 reference 的 crash/drgn 流程。

**核心要点：**

* 分层定位：沿 `syscall -> VFS -> FS -> Page Cache/Writeback -> blk-mq -> driver -> device` 拆解
* 先保现场，再分析；生产环境优先只读观测
* 列出可证伪假设，逐步排除
* 输出 RCA 或 Debug Report

## 3. Backport

**触发场景：** 社区 bugfix、stable patch、安全修复、特性 patch 回合到内部或发行版内核。

**参考文件：** [references/backport.md](references/backport.md)

**核心要点：**

* 记录 upstream SHA、Fixes tag、依赖链
* 对比目标分支的代码、config、API、结构体字段、锁模型
* 优先 `git cherry-pick -x`；手工适配必须说明偏离原因
* 验证编译、启动、reproducer、相关测试、debug config
* 不能以"能编译"作为 backport 成功标准

## 4. 测试验证

**触发场景：** 内核自测、存储回归、压力测试、性能测试、故障注入、回归报告。

**参考文件：** [references/testing.md](references/testing.md)

**核心要点：**

* 先定义测试意图：正确性、性能、稳定性、错误路径、并发竞态
* 设计测试矩阵：内核版本、FS、挂载参数、IO scheduler、设备、workload
* 选择合适工具：KUnit、kselftest、fstests、blktests、fio、fault injection
* 成功标准：无 WARN/BUG/KASAN/KCSAN/lockdep、fsck clean、数据校验通过

## 5. Patch Review

**触发场景：** 内部 patch、社区 patch、backport patch、性能优化 patch、bugfix patch。

**参考文件：** [references/patch-review.md](references/patch-review.md)

**可选外部 Skill 协作：** 当前环境若安装了 `sashiko`，深度自动化 review 可调用该 skill；未安装时直接按本 reference 执行源码审查。

**核心要点：**

* 审问题：commit message 是否完整
* 审拆分：一个 patch 一个逻辑变化
* 审正确性：上下文、锁、错误路径、生命周期、并发
* 审接口：uapi、sysfs、on-disk format 兼容性
* 审测试：reproducer、错误路径、debug config 覆盖
* 提交前：checkpatch、sparse、相关测试
* 需要深度自动化 review 且 `sashiko` 已安装时，可调用它进行多阶段审查

## 6. 性能优化

**触发场景：** IO 吞吐、p99/p99.9 尾延迟、锁竞争、CPU 开销、队列调度、内存回收干扰。

**参考文件：** [references/performance-optimization.md](references/performance-optimization.md)

**核心要点：**

* 定义明确指标：IOPS、BW、p50/p99/p99.9、CPU usage
* 建立稳定基线：固定硬件、内核、config、workload
* 宏观定位：CPU、IO、内存、调度、锁、设备
* 微观剖析：火焰图、perf lock、tracepoints、blktrace/btt
* 设计最小变更：减少热路径锁、批量化、per-cpu 化
* A/B 验证：多轮运行、完整百分位分布

## 文档沉淀

适用：Design Doc、Debug Report、RCA、Test Plan、Test Report、OnCall 通告。

**参考文件：** [references/document-templates.md](references/document-templates.md)

**通用要求：**

* 元信息完整：标题、作者、日期、内核版本、硬件、关键 config
* 源码引用使用 `path/to/file.c:function()` 或 `path/to/file.c:Lx`
* 命令、脚本、配置必须可复制执行
* 区分 [Fact]、[Inference]、[Hypothesis]、[Conclusion]
* 缺失数据写 `Pending: 需要采集 ...`，不要删除字段

## Output Contracts

根据任务类型输出对应产物：

* **开发实现：** 最小 patch / 设计说明 / 错误路径说明 / 测试证据 / 风险和回滚说明
* **问题定位：** RCA / Debug Report / 临时规避方案 / 根治 patch / 回归测试建议
* **Backport：** backport commit / 依赖说明 / 冲突说明 / 测试报告 / 回滚方案
* **测试验证：** Test Plan / Test Report / 原始日志与数据路径 / 失败用例跟踪
* **Patch Review：** review comments / open questions / Reviewed-by 条件
* **性能优化：** Perf Report / 原始数据 / patch / A/B 结果 / 适用范围与副作用
