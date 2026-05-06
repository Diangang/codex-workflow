# Patch Review Reference

本文件为内核存储与 I/O 子系统的 Patch Review 工作提供详细方法论。适用场景：内部 patch、社区 patch、backport patch、性能优化 patch、bugfix patch、patch series。

## Primary References

* Linux kernel patch 提交：`https://docs.kernel.org/process/submitting-patches.html`
* Patch 提交 checklist：`https://docs.kernel.org/process/submit-checklist.html`
* Coding style：`https://docs.kernel.org/process/coding-style.html`
* Sparse：`https://docs.kernel.org/dev-tools/sparse.html`
* Checkpatch：`https://docs.kernel.org/dev-tools/checkpatch.html`
* GFP masks from FS/IO context：`https://docs.kernel.org/core-api/gfp_mask-from-fs-io.html`

## 1. Review 总体流程

### 1.1 Review 优先级

按以下顺序进行 Review，前面的问题可以阻塞后面的 Review：

```
1. 问题是否成立 (commit message)
   ↓
2. 拆分是否合理 (patch structure)
   ↓
3. 正确性 (correctness)
   ↓
4. 生命周期 (object lifetime)
   ↓
5. 并发安全 (concurrency)
   ↓
6. 接口兼容 (ABI/API)
   ↓
7. 性能影响 (performance)
   ↓
8. 测试覆盖 (testing)
```

### 1.2 Review 结论分级

| 级别 | 含义 | 动作 |
|---|---|---|
| **Critical** | 正确性/安全/数据损坏风险 | 阻塞合入，必须修复 |
| **Major** | 语义错误/生命周期/并发问题 | 要求修复 |
| **Minor** | 编码风格/可读性/非关键路径 | 建议修复 |
| **Nit** | 纯格式/命名偏好 | 可选修复 |
| **Question** | 不确定正确性，需要解释 | 等待作者回复 |

## 2. 审 Commit Message

### 2.1 必须包含

1. **Context**：为什么需要这个变更，当前行为是什么
2. **Problem**：具体问题描述，how to reproduce
3. **User-visible Impact**：对用户/业务的影响
4. **Solution**：修复方案及其原因
5. **Testing**：如何验证修复
6. **Fixes tag**（bugfix 必须）：`Fixes: <sha> ("original commit")`

### 2.2 常见问题

* 只描述了"做了什么"，没有说"为什么"
* 问题描述模糊，无法判断是否真实存在
* 缺少 Fixes tag（bugfix 场景）
* 缺少测试说明
* 缺少性能数据（性能优化场景）

## 3. 审 Patch 拆分

### 3.1 拆分原则

* **一个 patch 一个逻辑变化**
* 重构、准备工作、行为变更、修复必须分开
* Series 中每个 patch 可编译、可 bisect
* 不在功能 patch 中夹带无关代码清理

### 3.2 检查清单

| 检查项 | 标准 |
|---|---|
| 每个 patch 单独编译 | `make` 不报错 |
| 每个 patch 可 bisect | 中间态不引入 bug |
| 逻辑变化隔离 | 一个 patch 只做一件事 |
| prep patch 在前 | 先重构再修改行为 |
| 辅助函数先引入 | 先加函数再使用 |

## 4. 审正确性

### 4.1 上下文语义

| 检查项 | 关注点 |
|---|---|
| **可睡眠性** | 新代码是否在 atomic/spinlock/irq 下执行了可能睡眠的操作 |
| **GFP flags** | atomic/IRQ/softirq 不得使用可睡眠分配；FS/IO 递归风险优先检查 `memalloc_nofs_save()` / `memalloc_noio_save()` scope，而不是随手加 `GFP_NOFS`/`GFP_NOIO` |
| **锁匹配** | 获取和释放是否配对，错误路径是否释放 |
| **错误路径** | goto 清理标签是否正确，资源释放顺序 |
| **返回值** | errno 是否合理，调用者是否检查 |

### 4.2 常见正确性问题

```c
// 错误示例 1: atomic 上下文睡眠
spin_lock(&lock);
kmalloc(size, GFP_KERNEL);  // BUG: atomic context 中可能睡眠
spin_unlock(&lock);

// 错误示例 2: 错误路径未释放
ptr = kmalloc(size, GFP_KERNEL);
if (!ptr)
    return -ENOMEM;
ret = do_something();
if (ret)
    return ret;  // BUG: 没有 kfree(ptr)

// 错误示例 3: 锁未释放
mutex_lock(&lock);
if (error_condition)
    return -EIO;  // BUG: 没有 mutex_unlock
mutex_unlock(&lock);
```

### 4.3 Kconfig/编译组合

* `#ifdef CONFIG_XXX` 是否覆盖所有路径
* 模块编译 vs 内建编译
* 不同架构下的行为差异
* `IS_ENABLED()` vs `#ifdef`

### 4.4 存储路径专项

| 区域 | 必查问题 |
|---|---|
| VFS/FS | `inode`/`file`/`address_space` 生命周期，truncate/invalidate/writeback/mmap 并发 |
| fsync/journal | 是否改变 data/metadata 持久化语义，是否需要 flush/FUA |
| Page Cache | folio/page lock、dirty/writeback 标记、引用计数是否成对 |
| blk-mq | request 合并、tag、budget、timeout、completion、queue freeze/quiesce 是否正确 |
| driver | reset/timeout/error recovery 是否完成或失败所有 inflight request |
| stacked device | clone bio/request 的完成顺序、错误传播和 original/clone 生命周期 |

## 5. 审生命周期

### 5.1 对象生命周期检查

| 对象类型 | 关注点 |
|---|---|
| **bio** | submit 后不可再访问、endio 回调释放 |
| **request** | blk-mq 生命周期、complete 后不可访问 |
| **inode** | iget/iput 配对、evict 后不可访问 |
| **folio/page** | get/put 配对、lock 语义 |
| **sk_buff** | consume 后不可再访问 |
| **kobject** | kref 引用计数、release 回调 |

### 5.2 引用计数检查

```
分配/获取 → 引用增加 → 使用 → 引用减少 → 释放
```

检查点：
* get/put 是否配对
* 错误路径是否 put
* 跨线程传递时是否额外 get
* RCU 保护期内是否安全

### 5.3 RCU 相关

* `rcu_read_lock()` / `rcu_read_unlock()` 配对
* `rcu_dereference()` 在 rcu read-side 使用
* `rcu_assign_pointer()` 在更新侧使用
* `synchronize_rcu()` / `call_rcu()` 选择
* grace period 完成前不释放对象

### 5.4 workqueue/timer/completion

* workqueue item 引用的对象在 cancel 前是否存活
* timer 回调访问的对象在 del_timer_sync 前是否存活
* completion 使用者是否可能在 complete() 后访问 completion 结构

## 6. 审并发安全

### 6.1 锁顺序

* 是否引入新的锁序？是否与已有锁序冲突？
* 是否可能导致 ABBA 死锁？
* nested lock 是否正确使用 `lockdep_set_class`？

### 6.2 内存屏障

* `smp_wmb()` / `smp_rmb()` / `smp_mb()` 使用是否正确
* `WRITE_ONCE()` / `READ_ONCE()` 是否覆盖共享变量
* `atomic_*` 操作的内存序是否满足需求

### 6.3 preempt/irq 语义

* `preempt_disable()` / `preempt_enable()` 配对
* `local_irq_save()` / `local_irq_restore()` 配对
* per-cpu 变量访问是否禁抢占

### 6.4 数据竞争检查

| 模式 | 风险 | 验证 |
|---|---|---|
| 读写无锁 | KCSAN 报告 | 添加 READ_ONCE/WRITE_ONCE 或锁 |
| TOCTOU | 检查和使用间状态变化 | 在同一个临界区内完成 |
| 并发初始化 | 多线程同时初始化 | once/初始化锁 |

## 7. 审接口兼容

### 7.1 用户可见接口

| 接口类型 | 兼容性要求 |
|---|---|
| **uapi (syscall/ioctl)** | 绝对不能破坏，只能扩展 |
| **sysfs** | 不能删除已有文件，语义不能改变 |
| **procfs** | 格式变更需要版本控制 |
| **debugfs** | 相对宽松，但仍需注意用户工具 |
| **tracepoint** | 格式变更会影响 BPF 程序 |
| **module 参数** | 不能删除或改变语义 |
| **on-disk format** | 需要兼容性和升级路径 |

### 7.2 on-disk format 变更

如果 patch 修改了 on-disk format：
* 旧内核能否读取新格式？
* 新内核能否读取旧格式？
* 升级路径是什么？
* 需要 fsck 变更吗？
* feature flag/incompat flag 设置是否正确？

## 8. 审性能影响

### 8.1 热路径检查

| 检查项 | 影响 |
|---|---|
| 新增锁 | 竞争、cacheline bounce |
| 新增分配 | GFP_KERNEL 可能触发回收 |
| 新增同步等待 | 延迟增加 |
| 新增 printk/trace | IO 影响 |
| 循环/遍历 | O(n) 复杂度 |

### 8.2 性能数据要求

如果 patch 声称有性能改进：
* 必须有 before/after 数据
* 必须有完整百分位分布
* 必须说明测试环境和 workload
* 必须说明是否有回归

## 9. 审测试覆盖

### 9.1 测试要求

| patch 类型 | 测试要求 |
|---|---|
| **bugfix** | reproducer + 修复验证 |
| **新功能** | 功能测试 + 错误路径 |
| **性能优化** | A/B 数据 + 回归测试 |
| **重构** | 功能等价性验证 |

### 9.2 Debug Config 覆盖

Review 时确认作者是否在以下 config 下测试过：

```
CONFIG_KASAN=y
CONFIG_KCSAN=y
CONFIG_PROVE_LOCKING=y
CONFIG_DEBUG_ATOMIC_SLEEP=y
CONFIG_DEBUG_LIST=y
CONFIG_BUG_ON_DATA_CORRUPTION=y
```

## 10. 提交前检查（自己的 patch）

### 10.1 自动化检查

```bash
# checkpatch
scripts/checkpatch.pl --strict --codespell *.patch

# sparse
make C=2 CHECK=sparse

# 编译 warning
make W=1

# 获取 maintainer
scripts/get_maintainer.pl *.patch
```

### 10.2 格式检查

```bash
# 生成 patch
git format-patch -n HEAD~<n>

# 或使用 b4
b4 prep --edit-cover
b4 send

# 验证线程结构
# Series 应该有 cover letter (0/n)
# 每个 patch 有正确的 In-Reply-To
```

### 10.3 提交前验证

1. `checkpatch --strict` 无 error
2. 目标子系统 `make` 无 warning
3. 相关 fstests/blktests 无新增失败
4. KASAN/KCSAN/lockdep debug kernel 冒烟
5. dmesg clean
6. `get_maintainer.pl` 确认收件人

## 11. Review 输出格式

### 11.1 Review Comment 格式

```
[Critical/Major/Minor/Nit/Question] <位置>

<问题描述>
<风险说明>
<建议修改>
```

### 11.2 最终结论

* **Reviewed-by**：代码正确性、逻辑、并发都验证通过
* **Acked-by**：从子系统角度认可方案和方向
* **Tested-by**：实际运行验证通过
* **需要修改**：列出必须修复的问题
* **需要讨论**：列出 open questions

## 12. 红线规则

* **不因为 patch 小就跳过错误路径和并发检查**
* **不给未理解的 patch 贴 Reviewed-by**
* **tracepoint/uapi/on-disk format 变更必须要求文档和兼容性说明**
* **安全相关 patch 不在公开列表讨论漏洞细节**
* **性能 patch 没有数据不给 Reviewed-by**
