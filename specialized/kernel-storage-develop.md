# Kernel Storage 开发

专属 skill。用于 Linux kernel storage/I/O 开发、bugfix、错误路径、锁、生命周期、tracepoint、sysfs/procfs/uapi 或性能相关代码改动。

## 默认叠加

- `common/operating.md`
- `common/verification.md`
- 有测试面时叠加 `common/tdd.md`

## 深度参考

- `external/raw/kernel-storage-workflow/references/development.md`
- 需要紧凑检查清单时读 `external/raw/linux-storage/SKILL.md`
- 涉及 Lite/Linux 对齐时读 `specialized/linux-alignment.md`

## 入口数据

- kernel 版本、分支、upstream base、相关 commit。
- 相关 `.config`。
- 硬件与存储拓扑。
- workload、reproducer 或预期调用路径。

## 改动规则

- 编辑前先画调用链和对象生命周期。
- 检查 process/atomic/IRQ/softirq/RCU 上下文。
- 检查睡眠规则、GFP/reclaim、锁序、引用计数、completion 和错误回滚。
- 用户可见 ABI、tracepoint、on-disk format、sysfs/procfs 变更必须先评估兼容性。
- 不能用“能编译”代替行为验证。
