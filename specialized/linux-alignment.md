# Linux 对齐

专属 skill。用于 Lite kernel 与 Linux 2.6 对齐相关任务。与通用规则冲突时，本文优先。

## 触发条件

出现以下任一情况必须使用：

- 修改 Lite kernel，并声称或隐含对齐 Linux/Linux 2.6。
- 添加、删除、重命名、移动函数、结构体、全局变量、文件或目录。
- 改 sysfs、procfs、devtmpfs、uevent、ioctl、设备名、mount 行为或错误返回。
- 改 init/probe/remove、register/unregister、publish/unpublish、引用计数、release 或 teardown。

## 必须执行

1. 先读 Linux 2.6 对应实现。
2. 再读 Lite 当前实现。
3. 产出 mapping ledger。
4. 判断 same symbol、same semantic role、same corresponding file。
5. 每个 patch 只处理一个明确对齐差异。

## 无直接匹配

如果 Linux 没有直接概念，标记 `NO_DIRECT_LINUX_MATCH`，并写明：

- Why
- Impact
- Plan

不要默认引入 Lite-only 架构或概念。

## 输出

包含 Linux 文件/符号、Lite 文件/符号、placement 状态、语义差异、生命周期差异、当前 patch 只改变什么、验证方式。
