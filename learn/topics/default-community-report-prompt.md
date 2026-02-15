# 默认社区报告 Prompt（中文翻译与讲解）

## 定位

- Prompt 键：`PROMPTS["community_report"]`
- 文件：`nano_graphrag/prompt.py:63`
- 使用位置：`generate_community_report(...)`，`nano_graphrag/_op.py:639`

## 中文完整翻译

你是一个 AI 助手，帮助人类分析师执行通用信息发现。  
信息发现是指：在一个网络中，识别并评估与某些实体（例如组织和个人）相关的信息。

### 目标（Goal）

给定一个社区中的实体列表、它们之间的关系，以及可选的相关声明（claims），撰写一份该社区的综合报告。  
这份报告将用于帮助决策者了解与该社区有关的信息及其潜在影响。  
报告内容包括：社区关键实体概览、其法律合规性、技术能力、声誉、以及值得关注的声明。

### 报告结构（Report Structure）

报告应包含以下部分：

- TITLE：社区名称，能够代表其关键实体；标题应简短但具体。尽可能在标题中包含具有代表性的命名实体。
- SUMMARY：社区整体结构的执行摘要，说明实体之间如何关联，以及与这些实体相关的重要信息。
- IMPACT SEVERITY RATING：0-10 的浮点分数，表示社区内实体所带来 IMPACT 的严重程度。IMPACT 指社区的重要性评分。
- RATING EXPLANATION：用一句话解释该 IMPACT 严重度评分。
- DETAILED FINDINGS：5-10 条关于社区的关键洞见列表。每条洞见应有一个简短摘要，随后是多段解释性文本，并且解释必须遵循下方 grounding 规则。内容应尽可能全面。

以格式良好的 JSON 字符串输出，格式如下：

```json
{
  "title": <report_title>,
  "summary": <executive_summary>,
  "rating": <impact_severity_rating>,
  "rating_explanation": <rating_explanation>,
  "findings": [
    {
      "summary": <insight_1_summary>,
      "explanation": <insight_1_explanation>
    },
    {
      "summary": <insight_2_summary>,
      "explanation": <insight_2_explanation>
    }
    ...
  ]
}
```

### 依据约束规则（Grounding Rules）

不要包含任何缺乏支持证据的信息。

### 示例输入（Example Input）

文本：

```text
Entities:
id,entity,type,description
5,VERDANT OASIS PLAZA,geo,Verdant Oasis Plaza is the location of the Unity March
6,HARMONY ASSEMBLY,organization,Harmony Assembly is an organization that is holding a march at Verdant Oasis Plaza

Relationships:
id,source,target,description
37,VERDANT OASIS PLAZA,UNITY MARCH,Verdant Oasis Plaza is the location of the Unity March
38,VERDANT OASIS PLAZA,HARMONY ASSEMBLY,Harmony Assembly is holding a march at Verdant Oasis Plaza
39,VERDANT OASIS PLAZA,UNITY MARCH,The Unity March is taking place at Verdant Oasis Plaza
40,VERDANT OASIS PLAZA,TRIBUNE SPOTLIGHT,Tribune Spotlight is reporting on the Unity march taking place at Verdant Oasis Plaza
41,VERDANT OASIS PLAZA,BAILEY ASADI,Bailey Asadi is speaking at Verdant Oasis Plaza about the march
43,HARMONY ASSEMBLY,UNITY MARCH,Harmony Assembly is organizing the Unity March
```

示例输出：

```json
{
  "title": "Verdant Oasis Plaza 与 Unity March",
  "summary": "该社区围绕 Verdant Oasis Plaza 展开，它是 Unity March 的举办地点。该广场与 Harmony Assembly、Unity March 和 Tribune Spotlight 存在关系，这些实体都与该游行事件有关。",
  "rating": 5.0,
  "rating_explanation": "由于 Unity March 期间可能出现动荡或冲突，影响严重度评为中等。",
  "findings": [
    {
      "summary": "Verdant Oasis Plaza 是核心地点",
      "explanation": "Verdant Oasis Plaza 是该社区的核心实体，作为 Unity March 的举办地点存在。该广场是连接其他实体的共同纽带，体现其在社区中的重要性。该广场与游行事件的关联可能引发公共秩序问题或冲突，具体取决于游行性质及其引发的反应。"
    },
    {
      "summary": "Harmony Assembly 在社区中的角色",
      "explanation": "Harmony Assembly 是社区中的另一关键实体，是 Verdant Oasis Plaza 游行活动的组织者。Harmony Assembly 及其组织的游行可能成为潜在风险来源，具体取决于其目标以及引发的社会反应。理解 Harmony Assembly 与广场之间的关系对于把握该社区动态至关重要。"
    },
    {
      "summary": "Unity March 是重要事件",
      "explanation": "Unity March 是在 Verdant Oasis Plaza 举行的重要事件。该事件是社区动态的关键因素，也可能成为潜在风险来源，具体取决于游行性质及其引发的社会反应。理解游行与广场之间的关系对于把握社区动态至关重要。"
    },
    {
      "summary": "Tribune Spotlight 的角色",
      "explanation": "Tribune Spotlight 正在报道发生在 Verdant Oasis Plaza 的 Unity March。这表明该事件已受到媒体关注，可能放大其对社区的影响。Tribune Spotlight 的作用可能会显著影响公众对事件及相关实体的认知。"
    }
  ]
}
```

### 真实数据（Real Data）

请使用以下文本作答。不要在答案中编造内容。

文本：

```text
{input_text}
```

报告应包含以下部分：

- TITLE：社区名称，能够代表其关键实体；标题应简短但具体。尽可能在标题中包含具有代表性的命名实体。
- SUMMARY：社区整体结构的执行摘要，说明实体之间如何关联，以及与这些实体相关的重要信息。
- IMPACT SEVERITY RATING：0-10 的浮点分数，表示社区内实体所带来 IMPACT 的严重程度。IMPACT 指社区的重要性评分。
- RATING EXPLANATION：用一句话解释该 IMPACT 严重度评分。
- DETAILED FINDINGS：5-10 条关于社区的关键洞见列表。每条洞见应有一个简短摘要，随后是多段解释性文本，并且解释必须遵循下方 grounding 规则。内容应尽可能全面。

以格式良好的 JSON 字符串输出，格式如下：

```json
{
  "title": <report_title>,
  "summary": <executive_summary>,
  "rating": <impact_severity_rating>,
  "rating_explanation": <rating_explanation>,
  "findings": [
    {
      "summary": <insight_1_summary>,
      "explanation": <insight_1_explanation>
    },
    {
      "summary": <insight_2_summary>,
      "explanation": <insight_2_explanation>
    }
    ...
  ]
}
```

依据约束规则：

不要包含任何缺乏支持证据的信息。

## 讲解

### 这个 prompt 的角色

它不是做“抽取”，而是做“社区级总结与风险解读”。

在流程中：

1. `community_schema` 先产出社区结构
2. `_pack_single_community_describe` 把单社区打包成输入文本
3. `community_report` prompt 让 LLM 输出标准 JSON 报告

调用入口：`nano_graphrag/_op.py:625`

### 为什么要严格 JSON 输出

后续代码会把模型返回转成 JSON（`convert_response_to_json_func`）并存入 `community_reports`，供 global/local 查询使用。

相关位置：

- 解析：`nano_graphrag/_op.py:659`
- 存储：`nano_graphrag/_op.py:697`

### 为什么有 rating

`rating` 是社区“影响严重度/重要度”的显式数值信号，后续可用于过滤和排序（例如 global 查询的最小评分阈值）。

相关位置：`nano_graphrag/_op.py:1050`

## 为什么会觉得 `rating` 很“怪”

这种感觉是合理的。`rating` 的特点是：

1. 它是 LLM 生成的启发式分数，不是可验证真值。
2. 它是“社区级先验分”（偏全局重要性），不是“针对当前问题的相关性分”。
3. 不同数据集/提示词下，分数刻度未必可比，缺乏统一校准。

换句话说，`rating` 更像“软信号”而不是“硬指标”。

## 代码里它到底怎么用

1. local 路径：社区选择时作为次级排序键（主键是命中频次）
   - `nano_graphrag/_op.py:725`
2. global 路径：可按阈值过滤
   - `nano_graphrag/_op.py:1050`
3. global 路径：在 `(occurrence, rating)` 里作为次级排序键
   - `nano_graphrag/_op.py:1054`
4. global map 阶段：`rating` 会作为表格字段送入下一轮 LLM
   - `nano_graphrag/_op.py:991`

## 实践建议

1. 如果你不信任该分数，可把 `global_min_community_rating` 设为 `0`（默认就是 0），避免过滤有效社区。
2. 把 `rating` 当作“弱先验”看待，核心仍依赖原始证据（实体/关系/文本片段）。
3. 若要增强一致性，可改写 `community_report` prompt 给出更清晰评分 rubric。

### Grounding 规则的意义

“不要编造”是为了把报告约束在可追溯证据上，降低幻觉。  
配合输入里提供的实体、关系、社区上下文，形成“有证据支撑的总结”。
