# Prompt 工程

专属 skill。用于创建、改写、评估、结构化 prompt，以及把模糊需求转成可复制执行的提示词。

## 触发场景

- 用户要求写 prompt、改 prompt、优化 prompt。
- 用户询问该用什么 prompt 框架或提示词结构。
- 用户要把需求交给 Codex、Claude Code、Cursor、ChatGPT、图像模型或 agent workflow 执行。
- 用户已有输出，希望反推或恢复生成该输出的 prompt。

## 默认叠加

- 涉及事实、政策、版本、价格或时间敏感信息时叠加 `common/verification.md`。

## 深度参考

- 框架细节：`external/raw/prompt-architect/SKILL.md`
- OpenAI 风格模板：`references/openai-prompt-templates.md`

## 执行规则

1. 按任务意图选择框架：create、transform、reason、critique、recover、clarify、agentic。
2. 只问缺失的关键问题；可以合理假设时直接生成。
3. 输出必须包含可直接复制的最终 prompt。
4. 给 Codex 执行的 prompt 要明确工作目录、目标、约束、验证方式和停止条件。
5. 用户要求“OpenAI 官方建议/最新模板/复杂版/精简版/reasoning 模型模板”时，读取 `references/openai-prompt-templates.md`。

## 输出

默认顺序：

1. 选择的框架和原因。
2. 关键改动点。
3. 最终 prompt，放在独立代码块中。

当 `external/raw/prompt-architect/SKILL.md` 对输出顺序有更严格要求时，优先遵守原 skill。
