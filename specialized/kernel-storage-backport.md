# Kernel Storage Backport

专属 skill。用于将 upstream/stable/community kernel storage/I/O 修复、安全补丁或特性回合适配到目标 kernel 分支。

## 默认叠加

- `common/operating.md`
- `common/verification.md`

## 深度参考

- `external/raw/kernel-storage-workflow/references/backport.md`
- 涉及 Lite/Linux 对齐时读 `specialized/linux-alignment.md`

## 入口数据

- upstream commit SHA、Fixes tag、stable tag、邮件线程或补丁来源。
- 目标分支、upstream base、内部补丁。
- 目标分支 `.config`、API、结构体字段和锁模型差异。
- 原始 bug/reproducer、影响范围和回归风险。

## 规则

- 先确认依赖链。
- 对比源分支和目标分支语义。
- 优先保持 upstream 语义。
- 手工偏离必须说明原因、影响和验证。
- 不混入 cleanup、重构或优化。
- 不能以“能编译”作为 backport 成功标准。
