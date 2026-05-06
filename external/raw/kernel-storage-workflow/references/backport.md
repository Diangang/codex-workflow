# Backport Reference

本文件为社区 bugfix、stable patch、安全修复、特性 patch 回合到内部或发行版内核提供详细方法论。

## Primary References

* Linux stable 规则：`https://docs.kernel.org/process/stable-kernel-rules.html`
* Backporting and conflict resolution：`https://docs.kernel.org/process/backporting.html`
* Linux kernel patch 提交：`https://docs.kernel.org/process/submitting-patches.html`
* Patch 提交 checklist：`https://docs.kernel.org/process/submit-checklist.html`

## 1. Backport 准入评估

### 1.1 Stable 规则快速判断

普通 stable bugfix 回合至少满足：

* 修复已经进入 upstream，或明确说明为什么必须提前进入目标分支。
* 有真实用户影响，不能只是代码清理、重构或理论优化。
* patch 尽量小且明显正确，不引入新 feature、ABI 变化或大范围重构。
* 包含 `Fixes:` 或清楚说明受影响范围；适合时包含 `Cc: stable@vger.kernel.org`。
* 有可复现问题、测试证据或 maintainer 已接受的风险说明。

### 1.2 来源记录

每个 backport 必须记录：

| 字段 | 内容 |
|---|---|
| Upstream SHA | 完整 commit hash |
| Fixes tag | `Fixes: abc123 ("original commit")` |
| Stable tag | `Cc: stable@vger.kernel.org` |
| 子系统 maintainer | 原始 patch 的审核者 |
| 邮件线程 | LKML/子系统列表链接 |
| 用户可见影响 | 问题描述和影响范围 |
| 目标分支 | 内部内核版本/发行版内核 |

### 1.3 适用性判断

在开始 backport 前必须评估：

1. **问题是否存在于目标分支**：代码路径是否一致
2. **修复是否适用**：API、数据结构、锁模型是否匹配
3. **stable 规则合规性**（针对 stable 回合）：
   - 是否已在 upstream 合入
   - 是否有真实用户影响
   - 变更是否小且明显正确
   - 是否已经过测试
4. **特性回合额外要求**：
   - 需要设计文档和兼容性说明
   - 需要更广泛的测试覆盖
   - 需要评估性能影响和副作用

### 1.4 分类

| 类型 | 复杂度 | 要求 |
|---|---|---|
| **Simple bugfix** | 低 | cherry-pick clean，直接测试 |
| **Bugfix with context changes** | 中 | 适配代码差异，验证语义一致 |
| **Security fix** | 高 | 安全响应流程，披露范围控制 |
| **Multi-patch bugfix** | 高 | 完整依赖链回合 |
| **Feature backport** | 最高 | 设计文档、兼容性、测试矩阵 |

## 2. 依赖分析

### 2.1 查找依赖

```bash
# 冲突文件在目标分支 HEAD 与待回合 patch 父提交之间的变更
git log HEAD..<upstream-sha>^ -- <path>

# 只看冲突函数的历史
git log -L:'\<function\>':<path> HEAD..<upstream-sha>^

# 搜索特定语义变更，如字段赋值、函数调用、errno 改动
git log -G'regex' HEAD..<upstream-sha>^ -- <path>

# 在待回合 patch 的父提交上看冲突行由谁引入
git blame -L:'\<function\>' <upstream-sha>^ -- <path>

# 比较目标分支和 upstream 差异
git range-diff <target-base>..<target-head> <upstream-base>..<upstream-sha>
```

`git log` / `git blame` 的目的不是找到"能套上的上下文"，而是找到导致冲突的 commit，并判断它是必须一起回合的语义依赖，还是可以跳过的偶发上下文变化。

### 2.2 依赖链分类

| 依赖类型 | 描述 | 处理方式 |
|---|---|---|
| **编译依赖** | 接口/宏/类型定义 | 必须先回合或适配 |
| **语义依赖** | 逻辑前置条件 | 必须先回合 |
| **上下文依赖** | git apply 冲突 | 可手工适配 |
| **后续修复** | 原 patch 的 bugfix | 必须一起回合 |
| **偶发上下文** | whitespace、重命名、轻微重排且不改变语义 | 不应为了解冲突引入无关 patch |

### 2.3 Prerequisite vs. Incidental 判断

* 如果目标分支缺少被修复的函数、状态字段、锁模型或错误处理框架，通常是 prerequisite，先回合依赖或停止。
* 如果只是同一区域的注释、格式、变量重命名或上下文重排，通常是 incidental，手工适配目标分支语义即可。
* 如果 upstream 已把一段逻辑抽成 helper，而目标分支仍有多份展开代码，backport 可能需要把同一修复应用到多个位置；用 `git grep` 在目标分支和 upstream 分别搜索同类模式。
* 如果无法确认依赖性质，不要硬改冲突；阅读相关 changelog，必要时询问 patch 作者或 maintainer。

### 2.4 冲突处理原则

* **只解决必要差异**：不带入无关重构
* **锁模型差异**：目标分支锁语义不同时，必须确认新锁在正确上下文使用
* **引用计数差异**：确认 get/put 配对在目标分支仍正确
* **错误路径差异**：确认错误处理和资源释放在目标分支完整
* **tracepoint 参数差异**：tracepoint ABI 变更需要单独评估
* **完整函数审查**：冲突处之外的 error label、return/break/continue、cleanup 顺序也必须审

## 3. 执行步骤

### 3.1 Cherry-pick 流程

```bash
# 1. 创建工作分支
git checkout -b backport/<sha_short> <target-branch>

# 2. 优先使用 cherry-pick；-x 保留来源记录
git cherry-pick -x <upstream-sha>

# 3. 如果 patch 来自邮件，先找能 clean apply 的合适 base
#    在该 base 上 git am / b4 am 后形成 commit，再 cherry-pick 到目标分支
b4 am -g -3 <message-id-or-mbox>

# 4. 如果有冲突，先设置更清晰的冲突标记
git config merge.conflictStyle diff3

# 5. 解决冲突后检查 staged 和 unstaged diff
git diff HEAD
git diff --cached
git cherry-pick --continue

# 6. 编译验证
make -j$(nproc) O=build/

# 7. 启动验证
# (根据环境使用 QEMU 或真实物理机)

# 8. 功能验证
# 运行原始 reproducer
# 运行相关测试
```

若邮件 patch 不能直接套到目标分支，不要手改 `.rej` 文件硬套。优先找接近原 patch 的 base 让它 clean apply，再通过 git history 做 cherry-pick 冲突解析。

### 3.2 手工适配规范

当 cherry-pick 冲突无法自动解决时：

1. 记录所有偏离 upstream 的改动
2. 在 commit message 中说明每处适配的原因
3. 确保适配后的语义与 upstream 一致
4. 对锁、引用计数、错误路径做二次审查
5. 如果适配导致语义不确定，暂停并升级讨论

### 3.3 冲突解析工作流

1. 先读原 patch 的 changelog 和每个 hunk，回答"这个 hunk 为什么存在"。
2. 用 `git diff` 看 combined diff，用 `git diff HEAD` 或 `git diff --ours` 看当前分支到已解析结果的普通 diff。
3. 对复杂冲突可先保留目标分支代码，再按原 patch 意图手工应用最小变更。
4. 用 `git add -i` 分块 staging，逐步缩小未解决范围。
5. 对文件重命名冲突，先判断是否应一起回合 rename patch；小变更可手工套，大变更可尝试降低 rename detection 阈值：

```bash
git cherry-pick --strategy=recursive -Xrename-threshold=30 <upstream-sha>
```

6. 冲突解决后必须用 `git diff -W` / `git show -W` 审完整函数，尤其是 goto label、return/break/continue 和 cleanup 顺序。

### 3.4 Commit Message 格式

```
<original subject>

[ Upstream commit <full_sha> ]

<original commit message body>

[ 如有适配说明 ]
Conflicts:
  path/to/file.c: <冲突原因和处理方式>
  path/to/other.c: <为何不是 prerequisite，而是手工适配>

Signed-off-by: Your Name <email@example.com>
```

提交到 stable 时，老 stable 版本可能要求 `commit <mainline rev> upstream.` 形式；按目标 stable tree/maintainer 邮件要求为准。多个 active stable 版本需要分别提交、分别测试，不要复用一个测试结论。

## 4. 验证矩阵

### 4.1 必须验证项

| 验证阶段 | 内容 | 通过标准 |
|---|---|---|
| **单文件编译** | `make path/to/file.o O=build/` | 快速发现局部编译错误 |
| **完整编译** | `make O=build/` | 无 error，无新 warning/link error |
| **启动** | boot 到 userspace | dmesg clean，无 WARN/BUG |
| **原始 reproducer** | 问题不再复现 | reproducer 通过 |
| **相关测试** | fstests/blktests/fio | 无新增失败 |
| **Debug config** | KASAN/KCSAN/lockdep | 无新增告警 |
| **长稳** | 持续运行数小时 | 无异常、dmesg clean |

### 4.2 Debug Config 要求

```
CONFIG_KASAN=y
CONFIG_KASAN_GENERIC=y
CONFIG_KCSAN=y
CONFIG_PROVE_LOCKING=y
CONFIG_DEBUG_ATOMIC_SLEEP=y
CONFIG_DEBUG_LIST=y
CONFIG_DEBUG_PLIST=y
CONFIG_BUG_ON_DATA_CORRUPTION=y
```

### 4.3 与 upstream patch 对比

冲突解决后，必须把 backport patch 和 upstream 原 patch 做函数级对比：

```bash
# 对比单个 commit 的函数上下文 diff
git diff -W <upstream-sha>^- > /tmp/upstream.diff
git diff -W HEAD^- > /tmp/backport.diff
diff -u /tmp/upstream.diff /tmp/backport.diff

# 如有 colordiff，可做 side-by-side 对比
colordiff -yw -W 200 /tmp/upstream.diff /tmp/backport.diff | less -SR
```

对比重点：

* 函数参数是否把相似变量传错
* `goto` 是否跳到目标分支正确 cleanup label
* 新增 `return` / `break` / `continue` 是否绕过必要释放
* refactor 后 upstream 单点修复是否需要在目标分支多点应用
* 上下文里未修改但语义不同的行是否改变了 patch 效果

Runtime testing 后仍要做 patch-level review；无冲突、能编译、能启动都不能证明 backport 语义正确。

## 5. 红线规则

* **不能以"能编译"作为 backport 成功标准**
* **缺失语义依赖时不要硬改冲突**：先补依赖或说明无法安全回合
* **不知道冲突来源时不要提交**：必须说明导致冲突的 commit 以及处理理由
* **安全修复按安全流程处理**：披露范围和发布时间需控制
* **大型特性/重构/ABI 变化**：不按普通 bugfix 流程处理
* **on-disk format 变化**：需要独立评估兼容性和升级路径
* **目标分支缺失语义依赖**：不允许用"能编译"作为通过标准
* **信心不足要明说**：提交或汇报时标注未验证项，并请求相关 maintainer ack

## 6. 输出要求

完成 backport 后必须交付：

1. **Backport commit**：包含 upstream SHA 和适配说明
2. **依赖清单**：列出所有前置 patch 和后续修复
3. **冲突说明**：每处冲突的来源 commit、是否 prerequisite、处理方式
4. **函数级 diff 对比**：说明与 upstream patch 的语义等价点和有意偏离点
5. **测试报告**：验证矩阵结果；多个 stable 版本分别列结果
6. **风险评估**：副作用、性能影响、兼容性、未验证项和信心等级
7. **回滚方案**：如何安全回退
