## Why

<!--
解释此变更的动机。解决什么问题？为什么现在处理？

硬限制：50 ≤ 字元数 ≤ 1000（OpenSpec zod schema 会 validate）
- 太短：会收到 `Why section must be at least 50 characters` error
- 太长：会收到 `Why section should not exceed 1000 characters` error

建议结构：现状痛点 → 为什么现在处理 → 预期收益（各 1-2 句）
-->

## What Changes

<!--
描述将要变更的内容。具体说明新增 capability、修改或删除。

对于有明确前后对比的行为变更，使用 From/To 格式（markdown 无 inline diff）：

**<Section or Behavior Name>**
- From: <current state / requirement>
- To: <future state / requirement>
- Reason: <why this change is needed>
- Impact: <breaking / non-breaking, who's affected>

多个变更可重复此 block；纯新增或纯删除可用简单列表描述。
-->

## Capabilities

### New Capabilities
<!--
新增的 capability。将 <name> 替换为 kebab-case 标识符。
命名规则见 openspec/specs/README.md：使用复合名词（至少 2 个 word），
例如 `user-auth`、`data-export`、`api-rate-limiting`，不用纯单词。
Each creates specs/<name>/spec.md
-->
- `<name>`: <brief description of what this capability covers>

### Modified Capabilities
<!--
现有 capability 中 REQUIREMENTS 正在变更的（不仅是实现）。
仅当 spec 级别行为变更时才在此列出。每个需要一个 delta spec 文件。
使用 openspec/specs/ 中的现有 spec 名称。若无需求变更则留空。
-->
- `<existing-name>`: <what requirement is changing>

## Impact

<!-- Affected code, APIs, dependencies, systems -->
