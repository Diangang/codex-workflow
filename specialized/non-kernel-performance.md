# 非内核性能

专属 skill。用于应用、服务、CLI、前端、后端、工具或自动化脚本的性能分析和优化。

## 默认叠加

- `common/operating.md`
- `common/verification.md`

## 入口数据

- 运行环境、版本、配置、依赖服务。
- workload、数据规模、并发度、输入样本。
- baseline：延迟分布、吞吐、CPU、内存、I/O、网络、错误率。
- profiling、trace、日志或监控数据。

## 流程

1. 明确指标和成功标准。
2. 固定环境和 workload，建立可重复 baseline。
3. 定位瓶颈类别：CPU、I/O、网络、锁/并发、数据库、渲染、序列化、启动时间。
4. 每次只改一个主要变量。
5. 用 A/B 数据验证。
6. 记录副作用和回滚方式。
