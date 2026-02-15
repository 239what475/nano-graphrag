# 默认实体抽取 Prompt（中文翻译与讲解）

## 定位

- Prompt 键：`PROMPTS["entity_extraction"]`
- 文件：`nano_graphrag/prompt.py:194`
- 使用位置：`extract_entities(...)`，`nano_graphrag/_op.py:295`

## 中文翻译（主模板）

下面是该 prompt 的核心模板翻译（Goal/Steps/Real Data），保留占位符：

### 目标（Goal）

给定一段可能与当前任务相关的文本，以及一个实体类型列表，从文本中识别出这些类型的所有实体，并识别这些实体之间的所有关系。

### 步骤（Steps）

1. 识别所有实体。对每个实体提取：
- `entity_name`：实体名称，首字母大写（capitalized）
- `entity_type`：以下类型之一：`[{entity_types}]`
- `entity_description`：该实体属性和活动的完整描述

将每个实体格式化为：
`("entity"{tuple_delimiter}<entity_name>{tuple_delimiter}<entity_type>{tuple_delimiter}<entity_description>)`

2. 在步骤 1 识别出的实体中，找出所有“明确相关”的实体对 `(source_entity, target_entity)`。
对每个相关实体对提取：
- `source_entity`：源实体名称（必须来自步骤 1）
- `target_entity`：目标实体名称（必须来自步骤 1）
- `relationship_description`：解释为何这两个实体相关
- `relationship_strength`：关系强度的数值分数

将每条关系格式化为：
`("relationship"{tuple_delimiter}<source_entity>{tuple_delimiter}<target_entity>{tuple_delimiter}<relationship_description>{tuple_delimiter}<relationship_strength>)`

3. 用英文输出步骤 1 和步骤 2 的全部结果，作为一个单列表。列表项之间使用 `**{record_delimiter}**` 分隔。

4. 完成时输出 `{completion_delimiter}`。

### 实际输入（Real Data）

- `Entity_types: {entity_types}`
- `Text: {input_text}`
- `Output:`

## 讲解（为什么这样设计）

1. 结构化可解析：要求固定 tuple 格式 + 分隔符，后处理可以直接 `split`，不依赖模型输出 JSON 的稳定性。
2. 先实体后关系：关系必须基于已识别实体，减少“关系引用未知实体”的噪声。
3. 限定实体类型：通过 `{entity_types}` 约束抽取边界，避免类型泛化过度。
4. 强制 completion 标记：`{completion_delimiter}` 让解析器知道输出结束位置，便于截断与补抽循环。
5. 用英文输出：在多语言文本里也尽量统一实体描述风格，降低后续汇总复杂度。

## 相关默认值

- `DEFAULT_ENTITY_TYPES = ["organization", "person", "geo", "event"]`
- `DEFAULT_TUPLE_DELIMITER = "<|>"`
- `DEFAULT_RECORD_DELIMITER = "##"`
- `DEFAULT_COMPLETION_DELIMITER = "<|COMPLETE|>"`

位置：`nano_graphrag/prompt.py:324` 到 `nano_graphrag/prompt.py:327`

## 补抽提示（同一流程中的另外两个 prompt）

- `entiti_continue_extraction`：提示模型“上轮漏了很多实体，继续补充”
- `entiti_if_loop_extraction`：让模型回答 `YES/NO`，判断是否继续补抽

位置：`nano_graphrag/prompt.py:314` 到 `nano_graphrag/prompt.py:322`

